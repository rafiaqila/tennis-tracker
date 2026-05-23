# Tennis Tracker

A personal tennis data pipeline. Log sessions through a Telegram bot, store everything in BigQuery, and get AI coaching feedback after each session. Built on a GCP free tier VM, runs for under $1/month.

---

## What it does

- Logs tennis sessions through a conversational Telegram bot. No forms, no app installs, just chat.
- Stores structured session data in BigQuery across two tables: `SESSION` and `MATCH_RESULT`.
- Generates an AI coaching snapshot after every session using Gemini.
- Answers free-form questions about your game via the `/qa` command, with persistent memory across sessions.
- Schedules sessions in natural language ("Saturday 8am at Taman Tun") and adds them to Google Calendar automatically.
- Sends a pre-session brief every night before a scheduled session: weather forecast, recent form, and context from past sessions.
- Sends a weekly Sunday summary with performance trends.
- Opens a Telegram Mini App dashboard with four tabs: Overview, Progress, Matches, and Activity.

---

## Screenshots

> **Bot conversation flow**
> ![Bot conversation](screenshots/bot-conversation.png)

> **Pre-session brief**
> ![Pre-session brief](screenshots/pre-session-brief.png)

> **Dashboard: Overview tab**
> ![Dashboard overview](screenshots/dashboard-overview.png)

> **Dashboard: Progress tab**
> ![Dashboard progress](screenshots/dashboard-progress.png)

See the full build writeup on [LinkedIn](https://www.linkedin.com/posts/raqilhidayat_personal-tennis-assistant-with-n8n-ugcPost-7450342290138529792-EICl/).

---

## Architecture

```
Telegram Bot
    ↓
N8N (self-hosted on GCP e2-micro via Docker Compose)
    ↓
BigQuery  ←→  Gemini API
    ↓
Google Calendar API
Open-Meteo API
    ↓
Telegram Mini App Dashboard (Chart.js)
```

---

## Tech stack

| Component | Tool |
|---|---|
| Interface | Telegram Bot |
| Automation | N8N (self-hosted) |
| Data storage | Google BigQuery |
| AI | Gemini `gemini-2.0-flash` |
| Scheduling | Google Calendar API |
| Weather | Open-Meteo API (no key needed) |
| Frontend | Telegram Mini App + Chart.js |
| Infrastructure | GCP e2-micro + Docker Compose + nginx + Let's Encrypt |

---

## Bot commands

| Command | What it does |
|---|---|
| `/log` | Start a new session logging flow |
| `/insights` | AI summary of recent performance trends |
| `/qa` | Ask anything about your game in natural language |
| `/schedule` | Schedule a session ("Friday 7pm at Section 17") |
| `/dashboard` | Open the Mini App dashboard |

---

## Data model

### `SESSION`

Covers all session types: Match, Social Game, Coaching, Solo Drill.

```
session_id        STRING    REQUIRED
session_date      DATE      REQUIRED
session_type      STRING    REQUIRED
format            STRING    NULLABLE
location          STRING    NULLABLE
duration_min      INTEGER   NULLABLE
conditions        STRING    NULLABLE
warmup            STRING    NULLABLE
drills_focus      STRING    NULLABLE
serve             INTEGER   NULLABLE
forehand          INTEGER   NULLABLE
backhand          INTEGER   NULLABLE
volley            INTEGER   NULLABLE
footwork          INTEGER   NULLABLE
mental_focus      INTEGER   NULLABLE
energy_before     INTEGER   NULLABLE
energy_after      INTEGER   NULLABLE
discomfort        STRING    NULLABLE
discomfort_detail STRING    NULLABLE
mood              STRING    NULLABLE
went_well         STRING    NULLABLE
improve           STRING    NULLABLE
notes             STRING    NULLABLE
return_rating     INTEGER   NULLABLE
created_at        TIMESTAMP REQUIRED
```

### `MATCH_RESULT`

Match-specific data, linked to `SESSION` via `session_id`.

```
match_id       STRING    REQUIRED
session_id     STRING    REQUIRED
opponent_name  STRING    NULLABLE
opponent_level STRING    NULLABLE
score          STRING    NULLABLE
result         STRING    NULLABLE
tactical_focus STRING    NULLABLE
created_at     TIMESTAMP REQUIRED
```

---

## Infrastructure

- **VM:** GCP e2-micro (us-central1), always-free tier
- **Stack:** Docker Compose running N8N + PostgreSQL + nginx
- **Domain:** DuckDNS with Let's Encrypt SSL
- **Timezone:** Asia/Kuala_Lumpur
- **Dashboard URL:** `https://raqil-n8n.duckdns.org/dashboard`
- **BigQuery region:** asia-southeast1 (Singapore)

---

## N8N workflow design

The session logging workflow uses a centralised dispatcher pattern. One `Send Next Question` Code node handles all question routing. All 19+ handler nodes connect independently into this single dispatcher. State persists between webhook calls via `$getWorkflowStaticData('global')` keyed by `chatId`. A Switch node with 25+ rules routes each incoming message to the right handler.

This keeps the workflow readable without spaghetti connections.

---

## Why these tools

**BigQuery over Snowflake.** Snowflake costs a minimum of roughly $25/month even for personal use. BigQuery is free at this data volume.

**Gemini over Vertex AI.** Vertex AI is overkill for a small personal dataset. The Gemini API via N8N costs less than $1/month with full control over prompts.

**Two tables, not three.** `SESSION` covers all session types. `MATCH_RESULT` stores match-specific data only. Clean separation without over-engineering.

**Self-hosted N8N on GCP.** Avoids the N8N Cloud subscription entirely.

---

## Running your own version

This is a personal project, not a plug-and-play template. But if you want to set up something similar, here is what you need:

1. A GCP project with BigQuery enabled and a service account with Data Editor + Job User roles
2. A Telegram bot token from [@BotFather](https://t.me/BotFather)
3. A Gemini API key
4. A Google Calendar OAuth2 credential
5. N8N self-hosted, either locally with ngrok or on a VM with a proper domain
6. Import the workflow JSON and configure credentials in N8N
7. Create the BigQuery dataset and tables using the schemas above

---

## Repo structure

```
tennis-tracker/
├── README.md
├── screenshots/
├── n8n/
│   └── workflow.json           (credentials stripped)
├── bigquery/
│   ├── schema_session.json
│   └── schema_match_result.json
├── infra/
│   └── docker-compose.yml      (secrets removed)
└── dashboard/
    └── index.html
```

---

Built by Raqil. Recreational tennis player, data person, tracks too many things.
