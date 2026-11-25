# ReachInbox – Full-stack Email Job Scheduler

A production-style email scheduler: an Express + BullMQ backend that schedules and sends emails through Ethereal SMTP with per-sender throttling, hourly rate limits, Slack alerts and Elasticsearch search, plus a React dashboard with Google login.

```
reachinbox-scheduler/
├── backend/            Express + TypeScript API, BullMQ worker, Drizzle ORM (Postgres)
│   ├── src/
│   │   ├── config/     env validation (zod)
│   │   ├── db/         schema, client, migrations runner
│   │   ├── queue/      BullMQ queue definition
│   │   ├── services/   scheduler, rateLimiter (Redis Lua), mailer, search (ES), slack
│   │   ├── workers/    email worker (idempotent processor)
│   │   ├── routes/     auth (Google OAuth), emails/campaigns, slack OAuth
│   │   ├── index.ts    API entry (optionally runs the worker in-process)
│   │   └── worker.ts   standalone worker entry (scale horizontally)
│   └── drizzle/        SQL migrations (applied automatically on boot)
├── frontend/           React 19 + Vite + Tailwind + TanStack Query
│   └── src/
│       ├── api/        typed fetch client + endpoints
│       ├── components/ ui/ (Button, Field, Modal, Card…), layout/, emails/, compose/, slack/
│       ├── hooks/      useAuth, queries (react-query), useDebounce
│       ├── pages/      Login, Emails (Scheduled/Sent), Compose, 404
│       ├── types/      API response types
│       └── utils/      formatting, CSV lead parsing
├── samples/leads_sample.csv
└── docker-compose.yml  Postgres, Redis (AOF), Elasticsearch (+ optional api/worker containers)
```

---

## 1. Quick start

**Prerequisites:** Node 20+, Docker.

```bash
# 1. Infra
docker compose up -d            # postgres:5432, redis:6379 (AOF on), elasticsearch:9200

# 2. Backend
cd backend
cp .env.example .env            # fill GOOGLE_* (and SLACK_* for alerts), see section 2
npm install
npm run dev                     # migrations run automatically; http://localhost:4000

# 3. Frontend (new terminal)
cd frontend
npm install
npm run dev                     # http://localhost:5173
```

Open http://localhost:5173, sign in with Google, click **Compose New Email**, upload `samples/leads_sample.csv` and schedule.

| URL | What |
|---|---|
| http://localhost:5173 | Dashboard |
| http://localhost:4000/admin/queues | **Live Bull Board** (queue, delayed/active/completed jobs) |
| http://localhost:4000/health | Health check |

**Ethereal senders:** on first boot, the backend creates `ETHEREAL_AUTO_SENDER_COUNT` (default 3) Ethereal test accounts and stores them in Postgres, so restarts reuse the same senders. To use fixed accounts, set `ETHEREAL_SENDERS=user1@ethereal.email:pass1,user2@ethereal.email:pass2`. Every sent email stores its Ethereal preview URL. Click a row in **Sent** and then **Open in Ethereal** to view the message.

**Scaling workers:** set `RUN_WORKER_IN_API=false` and run `npm run dev:worker` (or `npm run start:worker` after build) in as many terminals or containers as you like. `docker compose --profile app up -d --build` starts the API plus **2 worker replicas** to show that the limits hold across instances.

---

## 2. OAuth setup

### Google login (required)
1. Google Cloud Console → APIs & Services → Credentials → **Create OAuth client ID** → *Web application*.
2. Authorized redirect URI: `http://localhost:4000/auth/google/callback`
3. Put the client ID and secret in `backend/.env` (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`).

Flow: `/auth/google` → Google consent (with a CSRF `state` cookie) → `/auth/google/callback` exchanges the code, verifies the ID token, upserts the user, sets an httpOnly JWT cookie and redirects to `/dashboard`. The header shows the name, email and avatar, and has a logout option.

### Slack notifications (required for the rate-limit alert)
Slack requires **HTTPS** redirect URLs, so expose the backend with a tunnel:
```bash
ngrok http 4000          # → https://abc123.ngrok-free.app
```
1. https://api.slack.com/apps → **Create New App** → *From scratch*.
2. **OAuth & Permissions** → Redirect URL: `https://abc123.ngrok-free.app/slack/callback`. Bot scope: `incoming-webhook`.
3. **Manage Distribution** → activate public distribution (only needed to install in workspaces other than your own).
4. `.env`: `SLACK_CLIENT_ID`, `SLACK_CLIENT_SECRET`, `SLACK_REDIRECT_URI=https://abc123.ngrok-free.app/slack/callback`.

In the dashboard sidebar, click **Connect Slack**, pick a channel and approve. The backend exchanges the code (`oauth.v2.access`), stores the incoming-webhook URL for that user, and posts a confirmation message. Use **Test** to send a test message and **Disconnect** to remove it (the token is also revoked).

Behaviour:
- **Not connected:** rate-limit hits are simply not notified. Nothing crashes.
- **Connected later:** the worker reads the connection from Postgres at the moment the limit is hit, so alerts start immediately without a redeploy.
- **Dedupe:** one message per user, per limit scope, per hour window (Redis `SET NX`), not one per deferred job.

**Demo tip:** to trigger a live alert quickly, set `MAX_EMAILS_PER_HOUR_PER_SENDER=2` (or use a campaign hourly limit of 2), then schedule 10 emails. The Slack message arrives the moment the third email from a sender is attempted.

---

## 3. API

All `/api/*` routes need the session cookie.

| Method | Path | Description |
|---|---|---|
| GET | `/auth/google` · `/auth/google/callback` | Google OAuth |
| GET | `/auth/me` | Current user |
| POST | `/auth/logout` | Clears the session |
| POST | `/api/campaigns` | Schedule a batch: `{ subject, body, emails[], startTime, delayMs, hourlyLimit }` |
| GET | `/api/emails?view=scheduled\|sent&q=&page=&pageSize=` | List, or full-text search when `q` is set (Elasticsearch) |
| GET | `/api/stats` | Counts per tab |
| GET | `/api/senders` | Sender pool with live usage for this hour |
| GET / DELETE | `/api/slack` | Slack connection status / disconnect |
| POST | `/api/slack/test` | Send a test message |
| GET | `/slack/connect` · `/slack/callback` | Slack OAuth |
| — | `/admin/queues` | Bull Board UI (optional basic auth: `BULL_BOARD_USER` / `BULL_BOARD_PASSWORD`) |

---

## 4. How it works

### 4.1 Scheduling (no cron)
`POST /api/campaigns`:
1. Inserts one `campaigns` row and one `emails` row per unique recipient, in chunks of 1,000. `scheduled_at = startTime + i × delay`, and senders are assigned round-robin. A unique index on `(campaign_id, to_email)` prevents duplicate leads.
2. Adds one **BullMQ delayed job** per row with `delay = scheduled_at − now` and **`jobId = emails.id`**, using `addBulk` in chunks of 500.
3. Bulk-indexes the rows into Elasticsearch.

There is no cron, polling loop or `node-cron` anywhere. Redis's delayed sorted set is the timer.

### 4.2 Persistence across restarts
- **Delayed jobs live in Redis, not in Node memory.** Redis runs with **AOF (`appendfsync everysec`)** and `noeviction`. When the API or worker restarts, the jobs are still in the delayed set and fire at their original timestamps. Jobs that became due while the server was down run as soon as a worker starts.
- **Postgres is the source of truth.** On boot, `reconcilePendingJobs()` checks every row still `scheduled`, `deferred` or `sending` and re-creates any job missing from Redis (for example after a Redis wipe), using the remaining delay. Because `jobId = email id`, rows that already have a job are left alone, and sent or failed rows are never touched. Nothing restarts "from Day 1".
- **Graceful shutdown:** on SIGINT/SIGTERM, the worker finishes its in-flight jobs and then closes.

*Verified locally:* emails were scheduled 15 s ahead, the API was killed, and one job was deleted from Redis by hand. Nothing was sent while the API was down. After the restart, reconciliation restored the missing job and each of the 4 emails was sent **exactly once**.

### 4.3 Idempotency (never send twice)
Three layers:
1. **Queue level:** `jobId = emails.id`. BullMQ ignores an add with an existing id, so re-submits and reconciliation can't create duplicate jobs.
2. **DB claim:** the worker atomically runs `UPDATE emails SET status='sending' WHERE id=? AND status IN ('scheduled','deferred')` (or the row is `sending` with a claim older than 5 minutes, meaning a crashed worker). Only the worker whose UPDATE returns a row may send. Rows already `sent` or `failed` are skipped.
3. **Deterministic `Message-ID: <emailId@reachinbox.scheduler>`.** In the one unavoidable edge case (the process dies after SMTP accepted the mail but before the DB write), a retry carries the same Message-ID, which receiving servers use to de-duplicate.

### 4.4 Concurrency
- `WORKER_CONCURRENCY` (default 5): parallel jobs per worker process. Any number of worker processes can run.
- Parallel jobs are safe because every shared decision (claiming a row, counting a send, spacing sends) is an **atomic Postgres UPDATE or Redis Lua script**. No in-memory state is shared.
- `WORKER_LIMITER_MAX` / `WORKER_LIMITER_DURATION_MS` (default 10 jobs/s): BullMQ's **global, Redis-backed limiter**. It caps total job throughput across all workers, protecting SMTP and the DB under bursts.

### 4.5 Delay between sends
Two levels:
- **Campaign spacing:** the compose form's *Delay between emails* staggers `scheduled_at` (email *i* goes at `start + i × delay`).
- **Per-sender minimum gap:** `MIN_DELAY_BETWEEN_SENDS_MS` defaults to **2,000 ms (min 2 seconds between two sends from the same sender)**, enforced in Redis so it holds across all workers and campaigns. If a job arrives too early, it gets a **FIFO reservation**: a per-sender cursor hands out the next free slot (`max(cursor, nextAllowedAt) + gap`), and the job is moved to the delayed set until that slot. Early jobs therefore queue up in order instead of all retrying at the same instant.

### 4.6 Emails-per-hour rate limiting
Enforced by one **atomic Redis Lua script** (`services/rateLimiter.ts`) run right before each send. The counters are keyed by clock-hour window:

| Key | Limit | Config |
|---|---|---|
| `rl:sender:{senderId}:{hourWindow}` | per sender | `MAX_EMAILS_PER_HOUR_PER_SENDER` (default 200) |
| `rl:campaign:{campaignId}:{hourWindow}` | per campaign | *Hourly limit* field in the compose form |
| `rl:global:{hourWindow}` | all senders | `MAX_EMAILS_PER_HOUR` (0 = off) |

The script checks every limit and, only if all pass, increments all counters and sets the sender's next-allowed timestamp, in one atomic step. Counters expire after 2 hours. Since the check and the increment are a single Redis operation, two workers can never both take the last slot.

**When a limit is hit, jobs are never dropped or failed:**
1. The job is assigned a slot in the **next hour window**: `nextHourStart + (slot − 1) × gap`, where `slot = INCR rl:resched:{scope}:{id}:{nextHour}`. Overflowing jobs are released in the order they overflowed, spaced out rather than all firing at hh:00:00. Order is preserved as far as possible.
2. The same BullMQ job is moved back to the delayed set (`job.moveToDelayed` + `DelayedError`). This does not count as a failed attempt and does not create a new job, so idempotency still holds.
3. The DB row becomes `deferred` (shown as *Rate limited* in the UI), its `scheduled_at` is updated, and `deferrals` is incremented. Elasticsearch is updated too.
4. A Slack alert is sent (once per scope per hour).

### 4.7 Behaviour under load (1000+ emails at the same time)
- Scheduling 1,000+ recipients is a few chunked multi-row INSERTs plus `addBulk` calls. *Measured locally: 1,200 emails were accepted and queued in about 0.4 s.*
- When they all become due, workers pull them under the global limiter. Each one passes the Lua check until a limit is reached, and the rest are deferred into the next hour in FIFO order.
- *Measured locally* (2 senders, campaign limit 20/h, 1,200 emails due at once): exactly 20 sent, the rest moved to `deferred` with increasing slots in the next hour, 0 duplicates and 0 failed jobs. In each following hour, the next batch goes out under the same limits until the campaign finishes.
- **Trade-off:** each over-limit job is still picked up once per hour to be re-deferred, which costs one Lua call and one UPDATE. That is cheap and keeps the design simple and robust. Raise `WORKER_LIMITER_MAX` to process deferrals faster. An alternative is to compute all send times upfront at scheduling time, but that breaks when limits change or several campaigns share a sender. Checking just in time keeps the limits correct across campaigns.

### 4.8 Retries
SMTP errors are retried `JOB_ATTEMPTS` times (default 3) with exponential backoff (30 s base). Between retries the row goes back to `scheduled`. After the last attempt it becomes `failed` with `failed_reason`, which is shown in the Sent tab.

### 4.9 Search (Elasticsearch)
Index `emails` (recipient, subject, body, sender, status, user, dates). Documents are bulk-indexed when emails are created and partially updated on every status change. `GET /api/emails?q=` runs a `multi_match` (`bool_prefix`, with recipient and subject boosted) filtered by user and tab, then loads the rows from Postgres in relevance order. Postgres stays the source of truth. If Elasticsearch is down, search falls back to SQL `ILIKE`, and the search box shows an `ES` or `SQL` badge.

### 4.10 Bull Board
Mounted at `/admin/queues` and updates live: delayed, waiting, active, completed and failed jobs, with job data and timings. Deferred jobs move back to *delayed* with their new timestamp. You can add basic auth through env.

---

## 5. Frontend

- **Google login** → dashboard. The header shows the avatar, name and email, and has a logout dropdown.
- **Sidebar:** Compose button, **Scheduled** and **Sent** tabs with live counts, per-sender usage for this hour, and the Slack connect card.
- **Scheduled / Sent tables:** email, subject, scheduled or sent time, and a status badge (`Scheduled`, `Rate limited`, `Sending`, `Sent`, `Failed`). Includes skeleton loading, empty states, error with retry, debounced Elasticsearch search, pagination, and a detail modal with the Ethereal preview link. Data refreshes every 5 s.
- **Compose:** subject, body, and drag-and-drop CSV/TXT upload. Every email in any column is extracted and de-duplicated, and the count is shown. Also start time, delay between emails and hourly limit, with client-side validation and a summary card showing the estimated finish.
- **Code quality:** reusable UI primitives (`Button`, `Input`, `Textarea`, `Modal`, `Card`, `EmptyState`, `Avatar`, `StatusBadge`), one generic `EmailTable` driven by a column config for both tabs, typed API layer (`types/api.ts`), TanStack Query hooks, and toasts for errors and success.

> The Figma link wasn't included in the brief I received, so the UI follows ReachInbox's light, green-accented style (sidebar with Compose and tabs, list view, full-page composer). Styles are all Tailwind classes, so adjusting them to the exact Figma is quick.

---

## 6. Configuration reference

See `backend/.env.example`. Key throughput settings:

| Var | Default | Meaning |
|---|---|---|
| `WORKER_CONCURRENCY` | 5 | Parallel jobs per worker process |
| `MIN_DELAY_BETWEEN_SENDS_MS` | 2000 | Minimum gap between sends from the same sender |
| `MAX_EMAILS_PER_HOUR_PER_SENDER` | 200 | Hourly cap per sender |
| `MAX_EMAILS_PER_HOUR` | 0 | Optional global hourly cap (0 = off) |
| `WORKER_LIMITER_MAX` / `_DURATION_MS` | 10 / 1000 | Global BullMQ limiter |
| `JOB_ATTEMPTS` | 3 | SMTP retry attempts |
| `RUN_WORKER_IN_API` | true | Run the worker inside the API process |

## 7. Assumptions & trade-offs
- Hour windows are **clock hours (UTC)**, not sliding windows. They are simpler, cheap to key in Redis and easy to explain ("resumes at 14:00"). A sliding window would need a sorted set per sender.
- The **campaign hourly limit** counts per campaign. The **server cap** counts per sender across all campaigns. Both must pass.
- A send slot is consumed when it is granted, even if SMTP then fails. This is conservative and never exceeds a provider limit.
- JWT session in an httpOnly, SameSite=Lax cookie. The Vite dev server proxies `/api`, `/auth`, `/slack` and `/admin`, so the app is effectively same-origin.
#   R e a c h I n b o x _ S c h e d u l e r  
 