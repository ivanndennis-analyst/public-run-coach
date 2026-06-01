# Marathon Coach — System Architecture

> AI-powered marathon training coach built on a fully serverless stack.  
> Garmin → Strava → Supabase → Claude → Telegram · Melbourne, AU · Built Apr 2026

---

## What It Does

Every time Ivan finishes a run, the system automatically:

1. Receives the activity from Strava via webhook
2. Fetches the full HR stream and calculates time in each training zone
3. Computes hrTSS and updates the rolling CTL/ATL/TSB training load model
4. Sends the activity to Claude for personalised coaching analysis
5. Delivers a coaching note to Telegram within ~30 seconds of sync

A web dashboard provides deep analytics — projection, load, recovery, activities — accessible from any device.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR DEVICES                             │
│                                                                 │
│   ┌─────────────────┐         ┌─────────────────┐              │
│   │ Garmin FR 965   │ ──sync──▶│     Strava      │              │
│   │ GPS + HR watch  │         │ strava.com/api  │              │
│   └─────────────────┘         └────────┬────────┘              │
└────────────────────────────────────────│────────────────────────┘
                                         │ webhook POST
                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SUPABASE BACKEND                             │
│                                                                 │
│   ┌──────────────────────┐    ┌──────────────────────┐         │
│   │   strava-webhook     │    │   analyze-activity   │         │
│   │   Edge Function      │───▶│   Edge Function      │         │
│   │                      │    │                      │         │
│   │ • Validates event    │    │ • HR efficiency calc │         │
│   │ • Deduplicates       │    │ • Fetches context    │         │
│   │ • Fetches HR stream  │    │ • Calls Claude API   │         │
│   │ • Calculates zones   │    │ • Saves review to DB │         │
│   │ • Calculates hrTSS   │    │ • Sends to Telegram  │         │
│   │ • Updates daily_load │    └──────────────────────┘         │
│   └──────────────────────┘                                     │
│                                                                 │
│   ┌──────────────────────┐    ┌──────────────────────┐         │
│   │   weekly-summary     │    │  telegram-webhook    │         │
│   │   Edge Function      │    │  Edge Function       │         │
│   │                      │    │                      │         │
│   │ • Runs every Sunday  │    │ • Receives replies   │         │
│   │   6pm AEST (cron)    │    │ • Parses recovery    │         │
│   │ • Fetches week stats │    │   data naturally     │         │
│   │ • CTL/ATL/TSB status │    │ • Saves HRV/RHR/     │         │
│   │ • Injury risk check  │    │   sleep to DB        │         │
│   │ • Claude coaching    │    │ • Two-way coaching   │         │
│   │   focus for week     │    │   conversations      │         │
│   └──────────────────────┘    └──────────────────────┘         │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                  Postgres Database                      │  │
│   │                                                         │  │
│   │  strava_activities   training_plan    athlete_settings  │  │
│   │  daily_load          daily_recovery   telegram_messages │  │
│   └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │                                        │
          ▼                                        ▼
┌──────────────────┐                   ┌──────────────────────┐
│   Claude API     │                   │    Telegram Bot      │
│                  │                   │                      │
│ claude-sonnet    │                   │ @marathoncoach_id_bot│
│                  │                   │                      │
│ • Run coaching   │                   │ • Post-run coaching  │
│ • Weekly summary │                   │ • Weekly summaries   │
│ • Recovery       │                   │ • Two-way chat       │
│   interpretation │                   │ • Recovery logging   │
│ • Conversation   │                   │   via natural lang   │
│   history aware  │                   └──────────────────────┘
└──────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    WEB DASHBOARD                                │
│                    Next.js 13 · Vercel                          │
│                                                                 │
│   Dashboard     Activities    Projection    Load                │
│   ─────────     ──────────    ──────────    ────                │
│   Training      Activity      HR efficiency CTL/ATL/TSB        │
│   pulse         table with    model         chart               │
│   Week ahead    coaching      Weekly trend  Form state          │
│   Projections   review panel  Per-run       90-day history      │
│   Load summary  Time in zone  scatter                           │
│   Recovery      bars                        Recovery            │
│   snapshot                   Training Plan  ─────────           │
│                               ─────────────  HRV/RHR/Sleep     │
│                               Excel upload   TSB trend          │
│                               Coaching ctx   Zone split         │
│                               upload         Rest vs active     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Watch | Garmin Forerunner 965 | GPS, HR, cadence recording |
| Data pipeline | Strava API | Activity sync + webhook trigger |
| Backend | Supabase Edge Functions | Serverless TypeScript on Deno |
| Database | Supabase Postgres | All training + recovery data |
| AI | Anthropic Claude (claude-sonnet) | Coaching analysis + conversation |
| Notifications | Telegram Bot API | Post-run coaching + weekly summary |
| Frontend | Next.js 13 + Tailwind | Web dashboard |
| Hosting | Vercel | Frontend deployment |
| Design system | Google Stitch | Manrope + Inter, gold/steel palette |
| Auth | PIN lock (SHA-256 hash) | Client-side session protection |
| PWA | Web manifest + service worker | iPhone home screen install |

---

## Database Schema

### `strava_activities`
Stores every synced run with full metrics and coaching review.

| Column | Type | Description |
|---|---|---|
| strava_id | bigint | Unique Strava activity ID |
| name | text | Activity name from Strava |
| distance_km | numeric | Distance in kilometres |
| duration_seconds | int | Moving time |
| avg_hr | int | Average heart rate |
| elevation_gain | numeric | Elevation in metres |
| calories | int | Calories burned |
| date | timestamptz | Activity start date |
| raw_data | jsonb | Full Strava API response |
| claude_review | text | AI coaching note |
| z1_mins – z5c_mins | numeric | Minutes in each Friel zone |
| hr_tss | int | Calculated training stress score |

### `daily_load`
Rolling CTL/ATL/TSB training load model updated after every run.

| Column | Type | Description |
|---|---|---|
| date | date | Calendar date |
| hr_tss | numeric | Daily training stress score |
| atl | numeric | Acute Training Load (7-day fatigue) |
| ctl | numeric | Chronic Training Load (42-day fitness) |
| tsb | numeric | Training Stress Balance (form) |

### `daily_recovery`
Manually logged recovery metrics via Telegram natural language.

| Column | Type | Description |
|---|---|---|
| date | date | Calendar date |
| readiness_score | int | Subjective readiness 1–5 |
| hrv | numeric | Heart rate variability (ms) |
| rhr | int | Resting heart rate (bpm) |
| sleep_score | int | Garmin sleep score 0–100 |
| sleep_quality | text | Quality label (Good, Restless etc) |
| notes | text | Any additional context |

### `training_plan`
Imported from Excel — planned sessions across the marathon build.

| Column | Type | Description |
|---|---|---|
| week_number | int | Training week number |
| date | date | Session date |
| session_type | text | Type (Long Run, Tempo, Recovery etc) |
| planned_distance | numeric | Target distance in km |
| planned_duration | int | Target duration in minutes |
| planned_effort_zone | text | Target HR zone |

### `athlete_settings`
Single-row athlete profile used in every Claude prompt.

| Column | Type | Description |
|---|---|---|
| coaching_context | text | Full athlete background MD |
| goal_time | text | Target marathon time |

### `telegram_messages`
Conversation history used for context in Claude responses.

| Column | Type | Description |
|---|---|---|
| direction | text | inbound or outbound |
| message_text | text | Message content |
| created_at | timestamptz | Timestamp |

---

## HR Zone Model (Joe Friel 7-Zone)

Based on Lactate Threshold Heart Rate (LTHR).

| Zone | Name | BPM Range | Intensity Factor |
|---|---|---|---|
| Z1 | Recovery | < 129 | 0.65 |
| Z2 | Aerobic | 129–151 | 0.75 |
| Z3 | Tempo | 152–157 | 0.87 |
| Z4 | Sub-threshold | 158–162 | 0.95 |
| Z5a | Threshold | 163–166 | 1.00 |
| Z5b | VO2max | 167–171 | 1.05 |
| Z5c | Anaerobic | > 172 | 1.15 |

**hrTSS formula:** `(duration_hours × IF² ) × 100`  
where IF is the weighted intensity factor across zones.

---

## Training Load Model (CTL/ATL/TSB)

Based on the Banister Impulse-Response model used in TrainingPeaks.

```
ATL (Fatigue)  = prev_ATL + (TSS - prev_ATL) / 7
CTL (Fitness)  = prev_CTL + (TSS - prev_CTL) / 42
TSB (Form)     = CTL - ATL
```

| TSB Range | Form State | Meaning |
|---|---|---|
| > 25 | Very Fresh | Well rested — good for racing |
| 10–25 | Fresh | Recovered and ready |
| 0–10 | Productive | Fit and fresh — prime training |
| -10–0 | Fatigued | Manageable accumulated fatigue |
| -25– -10 | Very Fatigued | High fatigue — prioritise recovery |
| < -25 | Overreaching | Risk zone — reduce load now |

---

## Marathon Projection Model

HR efficiency-based projection weighted by recency and distance.

```
efficiency     = speed_km_per_min / avg_hr
marathon_speed = weighted_efficiency × (max_hr × 0.88)
projected_time = (42.195 / marathon_speed) × 60 × 1.08 (fatigue factor)
```

**Weighting:**
- Runs in last 8 weeks: full weight (1.0)
- Older runs: exponential decay `e^(-0.02 × days_over_56)`
- Distance weight: `min(distance_km / 20, 1.0)`

**Confidence:**
- High: 8+ qualifying runs in last 8 weeks
- Medium: 4–7 qualifying runs
- Low: fewer than 4 runs

---

## Data Flow — After Every Run

```
1. Run finishes on Garmin Forerunner 965
2. Auto-syncs to Garmin Connect (~1 min)
3. Auto-syncs from Garmin to Strava (~2 min)
4. Strava fires webhook POST to Supabase function URL
5. strava-webhook function:
   a. Checks for duplicate (deduplication guard)
   b. Fetches full activity from Strava API
   c. Fetches HR time-series stream from Strava API
   d. Calculates minutes in each of 7 HR zones
   e. Calculates hrTSS from zone distribution
   f. Saves activity + zone data to strava_activities
   g. Updates CTL/ATL/TSB in daily_load
   h. Invokes analyze-activity (fire and forget)
   i. Returns 200 immediately to prevent Strava retries
6. analyze-activity function:
   a. Calculates HR efficiency + marathon projection
   b. Fetches athlete coaching context
   c. Fetches last 6 conversation messages for continuity
   d. Sends structured prompt to Claude API
   e. Saves claude_review to strava_activities
   f. Sends formatted coaching note to Telegram
7. Coaching note arrives on phone within ~30 seconds
```

---

## Recovery Logging via Telegram

Send a natural language message to the bot and Claude extracts the data:

```
"HRV 58, resting HR 46, sleep 84 good, feeling 4/5"
```

The system parses: HRV=58, RHR=46, sleep_score=84, sleep_quality=Good, readiness=4  
Saves to `daily_recovery` table and replies with coaching context.

---

## Edge Functions

| Function | Trigger | Purpose |
|---|---|---|
| strava-webhook | Strava POST | Receive + process new activities |
| analyze-activity | Invoked by webhook | Claude coaching + Telegram delivery |
| weekly-summary | Cron Sun 6pm AEST | Weekly coaching summary to Telegram |
| telegram-webhook | Telegram POST | Two-way coaching + recovery logging |
| backfill-load | Manual HTTP POST | Recalculate CTL/ATL/TSB from history |
| backfill-reviews | Manual HTTP POST | Generate Claude reviews for old runs |
| backfill-activities | Manual HTTP POST | Re-sync historical Strava activities |

---

## Dashboard Pages

| Page | Purpose | Key Data |
|---|---|---|
| Dashboard | Morning briefing | KM pulse, week ahead, projection, load, recovery snapshot |
| Activities | Training log | All runs with coaching review + zone breakdown |
| Projection | Performance forecast | HR efficiency model, weekly trend, per-run scatter |
| Load | Physiological state | CTL/ATL/TSB chart, form state, 7d/21d/90d selector |
| Training Plan | Schedule management | Excel import, coaching context upload, week calendar |
| Recovery | Wellbeing tracking | HRV/RHR/sleep trends, zone split, TSB trend |
| Set Up | System configuration | Strava webhook, Claude API, Telegram bot setup |

---

## Build Approach

This entire system was built through conversation with Claude (claude.ai) — no local development environment, no IDE, no terminal access required.

**Tools used:**
- **Claude.ai** — all code written in conversation, architecture designed iteratively
- **GitHub** — code edited and committed directly via web interface
- **Vercel** — automatic deployments on every GitHub push to main
- **Supabase** — database, edge functions, secrets all managed via web UI
- **Hoppscotch** — browser-based API tool for webhook registration and testing
- **Telegram BotFather** — bot creation and token management

**Total build time:** ~6 weeks of iterative development  
**Lines of code:** ~4,000 across 15 files  
**Monthly running cost:** ~$2–5 (Supabase free tier + Claude API tokens)

---

## Secrets Reference

All secrets stored in Supabase Edge Function secrets. None are committed to the repository.

| Secret | Source | Notes |
|---|---|---|
| `STRAVA_CLIENT_ID` | strava.com/settings/api | App identifier — permanent |
| `STRAVA_CLIENT_SECRET` | strava.com/settings/api | Keep confidential |
| `STRAVA_ACCESS_TOKEN` | OAuth token exchange | Expires every 6 hours |
| `STRAVA_REFRESH_TOKEN` | OAuth token exchange | Used for auto-refresh |
| `STRAVA_VERIFY_TOKEN` | Self-created | Webhook verification shared secret |
| `ANTHROPIC_API_KEY` | console.anthropic.com | Charged per token |
| `TELEGRAM_BOT_TOKEN` | @BotFather in Telegram | Controls the coaching bot |
| `TELEGRAM_CHAT_ID` | Telegram getUpdates API | Personal chat identifier |

---

*Marathon Coach · Built Apr 2026 · Melbourne, VIC*  
*Garmin → Strava → Supabase → Claude → Telegram*
