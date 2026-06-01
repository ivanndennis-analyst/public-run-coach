# Marathon Coach — Setup Guide

> Get your own AI-powered marathon coach running in under 2 hours.  
> No coding experience required. Everything is done through web interfaces.

---

## Privacy & Data Notice

**Your data is yours. Completely.**

This is an open build — you deploy your own copy of the system using your own accounts. That means:

- **I (the developer) have zero access to your data.** You create your own Supabase database, your own Strava app, your own Telegram bot, your own Claude API key. I never see any of it.
- **Your training data lives in your Supabase account**, which you control entirely.
- **Your coaching conversations are private** — they go from your Telegram to your Supabase function to Anthropic's Claude API and back. No third party stores them.
- **Your Strava data is accessed only by your own API credentials.** The app reads your activities using keys you generate yourself.
- **You can delete everything at any time** by deleting your Supabase project and Vercel deployment.

The only external services your data touches are:
- **Supabase** (your own project) — database and functions hosting
- **Anthropic Claude API** (your own API key) — coaching analysis
- **Telegram** (your own bot) — message delivery
- **Vercel** (your own account) — web dashboard hosting

All of these are industry-standard services with their own privacy policies. None of them are controlled by me.

---

## What You'll Need

Before you start, create free accounts at these services:

| Service | URL | Cost |
|---|---|---|
| GitHub | github.com | Free |
| Supabase | supabase.com | Free tier sufficient |
| Vercel | vercel.com | Free tier sufficient |
| Anthropic | console.anthropic.com | Pay per use (~$2–5/month) |
| Telegram | telegram.org | Free |
| Strava | strava.com | Free |

**Estimated monthly cost:** $2–5 USD (almost entirely Claude API tokens at one run per day)

---

## Overview

You'll complete these steps in order:

```
1. Fork the GitHub repository
2. Set up Supabase (database + functions)
3. Connect your Strava account
4. Create your Telegram bot
5. Get your Claude API key
6. Deploy the dashboard on Vercel
7. Configure your athlete profile
8. Test the pipeline end-to-end
```

---

## Step 1 — Fork the Repository

1. Go to **github.com/ivanndennis-analyst/run-coach**
2. Click **Fork** in the top right
3. Choose your GitHub account as the destination
4. You now have your own copy of the codebase

---

## Step 2 — Set Up Supabase

Supabase is your database and serverless function host. Everything lives here.

### 2a — Create a project

1. Go to **supabase.com** and sign in
2. Click **New project**
3. Give it a name (e.g. `marathon-coach`)
4. Choose a region close to you
5. Set a database password — save this somewhere safe
6. Click **Create new project** and wait ~2 minutes

### 2b — Create the database tables

Once your project is ready, go to **SQL Editor** in the left sidebar and run each of these blocks one at a time.

**Activities table:**
```sql
create table strava_activities (
  id bigint generated always as identity primary key,
  strava_id bigint unique not null,
  name text,
  distance_km numeric(8,2),
  duration_seconds int,
  avg_hr int,
  elevation_gain numeric(8,1),
  calories int,
  date timestamptz,
  raw_data jsonb,
  claude_review text,
  z1_mins numeric(6,1),
  z2_mins numeric(6,1),
  z3_mins numeric(6,1),
  z4_mins numeric(6,1),
  z5a_mins numeric(6,1),
  z5b_mins numeric(6,1),
  z5c_mins numeric(6,1),
  hr_tss int,
  created_at timestamptz default now()
);

alter table strava_activities enable row level security;
create policy "Allow public read" on strava_activities for select using (true);
create policy "Allow public insert" on strava_activities for insert with check (true);
create policy "Allow public update" on strava_activities for update using (true);
```

**Training load table:**
```sql
create table daily_load (
  id bigint generated always as identity primary key,
  date date unique not null,
  hr_tss numeric(8,1),
  atl numeric(8,1),
  ctl numeric(8,1),
  tsb numeric(8,1),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

alter table daily_load enable row level security;
create policy "Allow public read" on daily_load for select using (true);
create policy "Allow public insert" on daily_load for insert with check (true);
create policy "Allow public update" on daily_load for update using (true);
```

**Recovery table:**
```sql
create table daily_recovery (
  id bigint generated always as identity primary key,
  date date unique not null,
  readiness_score int check (readiness_score between 1 and 5),
  hrv numeric(5,1),
  rhr int,
  sleep_score int,
  sleep_quality text,
  notes text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

alter table daily_recovery enable row level security;
create policy "Allow public read" on daily_recovery for select using (true);
create policy "Allow public insert" on daily_recovery for insert with check (true);
create policy "Allow public update" on daily_recovery for update using (true);
```

**Training plan table:**
```sql
create table training_plan (
  id bigint generated always as identity primary key,
  week_number int,
  date date,
  session_type text,
  planned_distance numeric(6,2),
  planned_duration int,
  planned_effort_zone text,
  created_at timestamptz default now()
);

alter table training_plan enable row level security;
create policy "Allow public read" on training_plan for select using (true);
create policy "Allow public insert" on training_plan for insert with check (true);
create policy "Allow public delete" on training_plan for delete using (true);
```

**Athlete settings table:**
```sql
create table athlete_settings (
  id bigint generated always as identity primary key,
  coaching_context text,
  goal_time text,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

alter table athlete_settings enable row level security;
create policy "Allow public read" on athlete_settings for select using (true);
create policy "Allow public insert" on athlete_settings for insert with check (true);
create policy "Allow public update" on athlete_settings for update using (true);

-- Insert a default row
insert into athlete_settings (goal_time) values ('3:30:00');
```

**Telegram messages table:**
```sql
create table telegram_messages (
  id bigint generated always as identity primary key,
  direction text,
  message_text text,
  created_at timestamptz default now()
);

alter table telegram_messages enable row level security;
create policy "Allow public read" on telegram_messages for select using (true);
create policy "Allow public insert" on telegram_messages for insert with check (true);
```

### 2c — Note your Supabase credentials

Go to **Settings → API** in your Supabase project and copy:

- **Project URL** — looks like `https://xxxxxxxxxxxx.supabase.co`
- **anon public key** — a long JWT token starting with `eyJ...`
- **service_role key** — another long JWT token (keep this private)

You'll need all three shortly.

### 2d — Deploy the Edge Functions

Go to **Edge Functions** in the left sidebar. You'll create 4 functions.

For each function click **New Function**, name it, and paste the code from the repository at these paths:

| Function name | Code location in repo |
|---|---|
| `strava-webhook` | `supabase/functions/strava-webhook/index.ts` |
| `analyze-activity` | `supabase/functions/analyze-activity/index.ts` |
| `weekly-summary` | `supabase/functions/weekly-summary/index.ts` |
| `telegram-webhook` | `supabase/functions/telegram-webhook/index.ts` |

After creating all four, go to **Settings → Edge Functions** and turn off **JWT verification** for all four functions.

---

## Step 3 — Connect Strava

### 3a — Create a Strava API application

1. Go to **strava.com/settings/api**
2. Click **Create & Manage Your App**
3. Fill in:
   - **Application Name:** Marathon Coach (or anything you like)
   - **Category:** Training
   - **Website:** your Vercel URL (you can update this later)
   - **Authorization Callback Domain:** `localhost`
4. Click **Save**
5. Note your **Client ID** and **Client Secret**

### 3b — Get an access token

You need to authorise your Strava account to allow the app to read your activities.

1. In your browser, go to this URL (replace `YOUR_CLIENT_ID` with your actual Client ID):
```
https://www.strava.com/oauth/authorize?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=http://localhost&approval_prompt=force&scope=activity:read_all
```
2. Click **Authorize** on the Strava page
3. You'll be redirected to a localhost URL that won't load — that's fine
4. Copy the `code=` value from the URL in your browser address bar

5. Now exchange that code for tokens. Open **Hoppscotch** (hoppscotch.io) and make a POST request to:
```
https://www.strava.com/oauth/token
```
With body (JSON):
```json
{
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "code": "THE_CODE_FROM_STEP_4",
  "grant_type": "authorization_code"
}
```
6. The response will contain `access_token` and `refresh_token` — copy both.

### 3c — Register the Strava webhook

In Hoppscotch, make a POST request to:
```
https://www.strava.com/api/v3/push_subscriptions
```
With body (form data, not JSON):
```
client_id: YOUR_CLIENT_ID
client_secret: YOUR_CLIENT_SECRET
callback_url: https://YOUR_SUPABASE_URL/functions/v1/strava-webhook
verify_token: marathoncoach2026
```

You should get back a subscription ID — make a note of it.

---

## Step 4 — Create Your Telegram Bot

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Choose a name for your bot (e.g. `My Marathon Coach`)
4. Choose a username ending in `bot` (e.g. `mymarathoncoach_bot`)
5. BotFather will give you a **bot token** — copy it

6. Now get your personal chat ID:
   - Search for **@userinfobot** in Telegram
   - Send it any message
   - It will reply with your **chat ID** — copy it

7. Register the webhook so Telegram sends messages to your Supabase function.  
In Hoppscotch, make a GET request to:
```
https://api.telegram.org/botYOUR_BOT_TOKEN/setWebhook?url=https://YOUR_SUPABASE_URL/functions/v1/telegram-webhook
```
You should get `{"ok":true,"result":true}` back.

---

## Step 5 — Get Your Claude API Key

1. Go to **console.anthropic.com** and sign in
2. Click **API Keys** in the left sidebar
3. Click **Create Key**
4. Give it a name (e.g. `marathon-coach`)
5. Copy the key — it starts with `sk-ant-...`
6. Add some credits under **Billing** — $10 will last several months at typical usage

---

## Step 6 — Add Your Secrets to Supabase

Go to **Supabase → Settings → Edge Function Secrets** and add each of these:

| Secret name | Value |
|---|---|
| `STRAVA_CLIENT_ID` | Your Strava Client ID |
| `STRAVA_CLIENT_SECRET` | Your Strava Client Secret |
| `STRAVA_ACCESS_TOKEN` | The access_token from Step 3b |
| `STRAVA_REFRESH_TOKEN` | The refresh_token from Step 3b |
| `STRAVA_VERIFY_TOKEN` | `marathoncoach2026` |
| `ANTHROPIC_API_KEY` | Your Claude API key from Step 5 |
| `TELEGRAM_BOT_TOKEN` | Your bot token from Step 4 |
| `TELEGRAM_CHAT_ID` | Your personal chat ID from Step 4 |

---

## Step 7 — Deploy the Dashboard on Vercel

1. Go to **vercel.com** and sign in with your GitHub account
2. Click **Add New → Project**
3. Find your forked `run-coach` repository and click **Import**
4. Under **Environment Variables**, add:

| Key | Value |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Your Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Your Supabase anon public key |

5. Click **Deploy**
6. Vercel will build and deploy your dashboard. Copy the URL it gives you (e.g. `your-project.vercel.app`)

### Set your PIN

The dashboard is protected by a PIN. To set your own:

1. Go to **emn178.github.io/online-tools/sha256.html**
2. Type your chosen 4-digit PIN
3. Copy the hash it generates
4. In your forked GitHub repo, open `lib/pin-auth.ts`
5. Replace the existing hash value with yours
6. Commit the change — Vercel will redeploy automatically

---

## Step 8 — Configure Your Athlete Profile

### 8a — Update your HR zones

The system uses Joe Friel 7-zone model based on your Lactate Threshold Heart Rate (LTHR).

To find your LTHR: run a 30-minute time trial at maximum sustainable effort. Your average HR for the last 20 minutes is approximately your LTHR.

Open `supabase/functions/strava-webhook/index.ts` in your repo and update the zone boundaries to match your LTHR:

```typescript
const ZONES = [
  { name: 'z1',  min: 0,   max: Math.round(LTHR * 0.81) },
  { name: 'z2',  min: ..., max: Math.round(LTHR * 0.89) },
  // etc
]
const LTHR = 166  // ← change this to your LTHR
```

The simplest approach: just change the `LTHR` constant and the zones will scale automatically.

### 8b — Upload your coaching context

In the dashboard, go to **Training Plan → Coaching Context** and upload a markdown file with your athlete background. This gets sent to Claude with every run analysis.

Include things like:
- Your running history and experience level
- Past injuries or physical limitations
- Goal race and target time
- Strengths and areas for improvement
- Training preferences and time constraints

Example:
```markdown
# Athlete Profile

Age: 35, Male
Experience: 5 years running, 3 marathons completed
Goal: Sub-3:30 at Melbourne Marathon, October 2026
Current fitness: Building base after 6 week injury layoff
Injury history: Left IT band issues when mileage exceeds 60km/week
Strengths: Strong aerobic base, good long run endurance
Weaknesses: Speed endurance, tempo pace
Preference: Morning runs, 5 days per week maximum
```

### 8c — Upload your training plan (optional)

If you have a structured training plan in Excel, go to **Training Plan → Training Plan Upload** and import it. The expected columns are:

- Week
- Date
- Session Type
- Planned Distance (km)
- Planned Duration (min)
- Planned Effort/Zone

### 8d — Set your goal time

Go to **Projection** in the dashboard, enter your goal marathon time, and click **Update Model**.

---

## Step 9 — Test the Pipeline

### Test Strava webhook
Do a short run and sync to Strava. Within 30 seconds you should receive a Telegram message from your bot with a coaching note.

If no message arrives, check **Supabase → Edge Functions → strava-webhook → Logs** for errors.

### Test Telegram two-way chat
Send a message to your bot, e.g.:
```
How should I structure my easy runs this week?
```
You should get a coaching reply based on your athlete context.

### Test recovery logging
Send your recovery metrics to the bot:
```
HRV 55, resting HR 48, sleep 82 good, feeling 4/5
```
Check **Supabase → Table Editor → daily_recovery** to confirm the data was saved.

### Test the dashboard
Open your Vercel URL, enter your PIN, and confirm:
- Dashboard loads with your data
- Activities page shows your runs
- Load page shows CTL/ATL/TSB (may need a few runs first)

---

## Backfilling Historical Data

If you have years of Strava history, you can backfill the training load model.

In Hoppscotch, make a POST request to:
```
https://YOUR_SUPABASE_URL/functions/v1/backfill-load
```
With headers:
```
Authorization: Bearer YOUR_SUPABASE_ANON_KEY
Content-Type: application/json
```
And body: `{}`

This will process all your historical activities and calculate CTL/ATL/TSB going back to your first run. It returns a summary of days processed and your current fitness numbers.

---

## Weekly Summaries

The system automatically sends a weekly coaching summary every Sunday at 8am UTC (adjust this to your timezone).

To change the schedule, go to **Supabase → Database → Extensions** and enable **pg_cron**, then run:
```sql
select cron.schedule(
  'weekly-summary',
  '0 8 * * 0',  -- change this cron expression for your timezone
  $$select net.http_post(
    url := 'https://YOUR_SUPABASE_URL/functions/v1/weekly-summary',
    headers := '{"Authorization": "Bearer YOUR_ANON_KEY"}'::jsonb
  )$$
);
```

To test immediately without waiting for Sunday, just POST to the weekly-summary function URL directly.

---

## Troubleshooting

**No Telegram message after a run**
- Check Supabase → Edge Functions → strava-webhook → Logs for errors
- Confirm your `STRAVA_ACCESS_TOKEN` hasn't expired (it refreshes automatically but may need a manual update first time)
- Confirm your Strava webhook subscription is still active

**Dashboard shows no data**
- Check Supabase → Table Editor → strava_activities to confirm data exists
- Check browser console (F12) for any errors
- Confirm your Vercel environment variables match your Supabase project URL and anon key exactly

**Claude not responding**
- Check your `ANTHROPIC_API_KEY` is correct and has credits
- Check Supabase → Edge Functions → analyze-activity → Logs

**Telegram bot not responding to messages**
- Confirm the webhook is registered (re-run the setWebhook URL from Step 4)
- Check Supabase → Edge Functions → telegram-webhook → Logs

**Strava access token expired**
- Check strava-webhook logs for `NEW_ACCESS_TOKEN:` entries
- Copy the new token value and update `STRAVA_ACCESS_TOKEN` in Supabase secrets

---

## Customisation

Once everything is running, common things to personalise:

- **HR zones** — update `LTHR` constant in strava-webhook function
- **Weekly km target** — update the `weekTarget` constant in `app/page.tsx`
- **Weekly hours target** — update the `hoursTarget` constant in `app/page.tsx`
- **Athlete name and goal** — update the hardcoded values in `components/sidebar-nav.tsx`
- **Coaching context** — update via the Training Plan page in the dashboard anytime

---

## Support

This is an open build shared for others to learn from and use. I'm happy to answer questions but can't guarantee ongoing support.

If something isn't working, the best starting point is always **Supabase → Edge Functions → [function name] → Logs** — the error messages are usually self-explanatory.

---

*Marathon Coach · Open Build · April 2026*  
*Built by Ivan Dennis · Melbourne, AU*
