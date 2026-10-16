# Tech Stack and Tools (approved by user: yes, 2026-10-16)

Decisions from the user: dev profile on Render, Turso and a free Redis; BullMQ for jobs; tag-based knowledge lookup (no vector DB); offline detection via BullMQ schedulers. The Heroku student credit is not available to the user, so Heroku-based plans in the source doc are dropped. Pinecone was named by the user but is not needed once retrieval is tag-based; it is dropped ([Q-14](05-assumptions-risks-questions.md#q-14) to confirm).

## Core stack

| Layer | Choice | Why | Alternatives considered |
|---|---|---|---|
| Runtime and framework | Node.js, NestJS, TypeScript | Source doc already assumes it; modules, guards and DI suit the auth, scoping and worker split | Fastify only (less structure) |
| HTTP and realtime | REST under `/api/v1`, socket.io for live events | Doc choice; socket.io works on a single instance; Render free wakes on a new WebSocket connection | SSE (simpler, but doc and frontend assume socket.io) |
| Database (dev) | Turso (libSQL) free | User decision; zero cost | Neon free Postgres (rejected: see [DR-2](#dr-2)), Render free Postgres (expires in 30 days) |
| Database (go-live) | Postgres (host TBD, [Q-1](05-assumptions-risks-questions.md#q-1)) | Same schema concept, more write headroom, `pgvector` available if ever needed | Stay on Turso paid |
| DB access | Repository layer plus Drizzle ORM (separate sqlite/turso and postgres schema files) | Drizzle supports both dialects; keeps SQL out of services | Kysely, raw SQL |
| Jobs and schedules | BullMQ with job schedulers | User decision; retries, backoff, rate limiting, cron with timezone | pg-boss (Postgres queue, no Redis; viable if Redis proves painful) |
| Redis (dev) | Render Key Value free (primary), Redis Cloud free (fallback) | Same-region private network; data is disposable by design ([DR-4](#dr-4)) | Upstash free (rejected: 500K commands/month vs BullMQ polling) |
| LLM | Groq: `llama-3.3-70b-versatile` for alert explanations, `llama-3.1-8b-instant` for narratives and overflow | Free, fast; split by token budget ([DR-6](#dr-6)) | Gemini free tier (prompts may be used by Google on the free tier per one secondary source: Unverified) |
| Knowledge retrieval | Tag-based table in the main DB | Alerts are structured; no embeddings or vector service needed | pgvector, Pinecone (dropped) |
| Hosting (dev) | Render free web service (one service) | User decision | Heroku (unavailable), Railway (usage-billed) |
| Keep-alive | External pinger calling `GET /healthz` every <= 10 min | Render sleeps free services after 15 idle minutes | None free and officially supported |

## External services and integrations

Checked 2026-10-16.

| Service | Purpose | Pricing | Limits and quotas | Caveats | Source |
|---|---|---|---|---|---|
| Render free web service | Runs NestJS app | $0 | Spins down after 15 min without inbound traffic; wakes on HTTP request or new WebSocket connection, about 1 min; 750 free instance hours per workspace per month; may restart at any time; local disk is ephemeral | One always-on service uses about 744 h, so a second free service will not fit. No official keep-alive; pinging is a workaround | Render docs |
| Render Key Value free | BullMQ backend | $0 | One free instance per workspace; no disk persistence; may restart at any time and lose data. 25 MB size from secondary sources: Unverified | Fine only because the DB is the source of truth ([DR-4](#dr-4)). Eviction policy must be `noeviction` for BullMQ; confirm it is settable (spike S2) | Render docs, BullMQ docs |
| Redis Cloud free (fallback) | BullMQ backend | $0 | 30 MB, 30 connections, 100 ops/sec; deleted after 14 days with no activity; no backups | Cross-provider latency; BullMQ polling counts as activity | Redis docs, Redis support |
| Turso free | Primary DB (dev) | $0 | 10M rows written and 500M rows read per month; queries fail with a BLOCKED error when a quota is exceeded | Writes include updates; index and delete accounting not documented (spike S1). Developer plan is $5.99 monthly or $4.99 billed yearly with 25M writes and $1 per extra million, not unlimited | Turso pricing, Turso billing docs |
| Groq free | Explanations, narratives, chat | $0 | `llama-3.3-70b-versatile`: 30 RPM, 1,000 RPD, 12K TPM, 100K TPD. `llama-3.1-8b-instant` (secondary source): 30 RPM, 14,400 RPD, 6K TPM, 500K TPD. Limits are per organization, not per key | The source doc's "14,400 req/day" applies to the 8B model only. Token limits, not request limits, are the constraint ([DR-6](#dr-6)) | Groq docs, secondary guides |
| External pinger | Keep Render awake, dead-man check | $0 | GitHub Actions schedule (can run late), cron-job.org or UptimeRobot. One source says UptimeRobot's free plan bars commercial use since Nov 2024: Unverified | Acceptable for dev and demo only | Render community, Odown, DEV |

## Decision records

- **<a id="dr-1"></a>DR-1 Dev hosting.** Context: Heroku credit unavailable. Options: Render free, Railway, Fly, VPS. Decision: Render free, as chosen by the user. Consequences: sleeping and restarts are normal events, so every design choice below assumes the process can vanish at any time and that "all devices offline" can put the service to sleep. Exit path: Dockerizable Node app, no Render-specific APIs, `render.yaml` only for deploy.
- **<a id="dr-2"></a>DR-2 Database.** Context: ingest is write-heavy (see [Estimated running cost and capacity](#estimated-running-cost-and-capacity)). Options: (a) Turso free, (b) Neon free Postgres, (c) Render free Postgres. Decision: Turso for dev. Neon free gives 100 CU-hours per project; at an assumed 0.25 CU minimum, 24/7 compute is about 186 CU-hours, so continuous 10-second ingest would exhaust it (Unverified: minimum CU not confirmed). Render free Postgres expires after 30 days. Consequences: dialect change at go-live. Mitigation: repository interfaces, portable schema (integer epoch milliseconds for time, text ids, no JSON operators in queries), both dialects in CI from SD1.
- **<a id="dr-3"></a>DR-3 Redis host.** Options: Render Key Value free, Redis Cloud free, Upstash free. Decision: Render Key Value first. Upstash's free tier allows 500K commands per month and BullMQ polls constantly (users report thousands of requests per hour when idle; anecdotal), so it would run out within days. Redis Cloud is the fallback if `noeviction` cannot be set on Render's free instance.
- **<a id="dr-4"></a>DR-4 Database is the source of truth, Redis is disposable.** Decision: readings and alerts are written to the DB before anything touches Redis. An alert is created with `explanation_status = pending`; the queue is only an accelerator. A reconciler re-enqueues anything pending on boot and every 5 minutes. Job schedulers are re-registered idempotently on every boot. Consequences: losing all of Redis causes delay, never data loss. Ingest has zero dependency on Redis.
- **<a id="dr-5"></a>DR-5 Retrieval.** Decision: tag-based KB lookup (user choice). The KB is a table keyed by sensor, level and composite condition; the best matches by specificity go into the prompt. Consequences: no embedding quota, no pinned embedding model, no vector DB. Revisit only if the KB grows past a few hundred entries or free-text matching is needed (pgvector in the same Postgres is the path).
- **<a id="dr-6"></a>DR-6 LLM routing and budget.** At roughly 1,500 tokens per explanation (assumption), the 70B free budget of 100K tokens per day is about 66 explanations. A daily narrative per device would consume most of it by itself at pilot scale (33 devices x about 2K tokens = 66K). Decision: explanations use 70B; narratives and chat overflow use 8B; a per-day ledger (`llm_usage_daily`) routes 70B -> 8B -> KB template. AI is never on the alert path. Alerts are throttled to one LLM call per (device, sensor, level) per 30 minutes.
- **<a id="dr-7"></a>DR-7 Ingest protocol.** Decision: batch endpoint (up to 60 readings), per-reading age offsets instead of device clock, `boot_id` + `seq` for idempotency, raw MQ135 ADC from the device with NH3 estimated server-side per calibration, HTTPS only. Why: ESP8266 has no battery-backed clock, TLS handshakes are slow on it (one report: 20 s with a full certificate store versus 2 s with a pinned fingerprint), Render cold starts drop requests for about a minute, and calibration must be changeable without reflashing. Firmware retries only on network errors, 5xx and 429, never on other 4xx.
- **<a id="dr-8"></a>DR-8 Scheduling and offline detection.** Decision: BullMQ job schedulers (user choice) for offline sweep, aggregates and retention, plus (a) an external pinger so the process stays awake, and (b) a catch-up pass on every boot. Why: an in-process scheduler cannot fire while the service is asleep, which happens exactly when every device is offline (for example a farm power cut). Consequence: offline alerts can be late by up to the pinger interval; with no pinger they fire on the next wake.

## Estimated running cost and capacity

**Dev profile: $0 per month** (all free tiers above).

**Turso write budget per device per month** (10 s interval = 8,640 readings per day, 259,200 per month):

| Component | Writes per device-month | Counted? |
|---|---|---|
| Reading inserts | 259,200 | Yes (inserts counted) |
| Index entry per insert | +259,200 | Unverified (a comparable product counts it) |
| Retention deletes (steady state) | +259,200 | Unverified |
| `last_seen` updates, throttled to 1 per minute | 43,200 | Yes (updates counted) |
| Aggregates, alerts, events | < 5,000 | Yes |

Free capacity = 10,000,000 / per-device writes: about **33 devices** if neither index entries nor deletes count, about **12 devices** if both count. The source doc's "38 devices" ignored `last_seen` updates, indexes and deletes. Spike S1 measures the real figure. Levers if short: lower sample rate to 30 s (3x capacity), shorter raw retention, Turso Developer plan ($4.99 to $5.99 per month, 25M writes).

**Go-live profile (indicative, not committed):** Render paid web service from $7 per month, Render Postgres from $6 per month, paid Key Value (secondary sources disagree, about $7 to $10) = roughly $20 to $25 per month at pilot scale. Verify at go-live ([Q-1](05-assumptions-risks-questions.md#q-1)). Groq paid-tier pricing must be fetched at that time.

Next: [Architecture](03-architecture.md)
