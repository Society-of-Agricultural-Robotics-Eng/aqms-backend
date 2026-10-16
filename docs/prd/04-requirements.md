# Requirements

Suggested order: M0, M1, M2, M3, M4, M5, M6, M7, M8, M9, then M10 only when go-live is decided. Reason: M1 and M2 prove the riskiest path (durable ingest and correct alerts) before any AI work; auth (M3) comes before realtime (M4) so WebSockets are never unauthenticated; the scheduler framework (M5) is needed before aggregates (M7). **First vertical slice:** M0 + M1 + M2 with the simulator, reading to alert row, no AI, no auth. Milestones M6 to M8 can run in parallel with M7 once M5 is done.

Sizes: S = under 1 day, M = 1 to 3 days, L = 3 to 5 days (one developer).

### Milestone M0: Foundations and spikes
<a id="m0"></a>
Goal: a deployable skeleton and the unknowns measured.
Definition of done: skeleton deployed to Render; spike results written into the [Assumption Ledger](05-assumptions-risks-questions.md#assumption-ledger).
Depends on: accounts ([Groundwork](01-overview.md#groundwork-settle-before-or-during-m0)).
Covers: [NFR-11](03-architecture.md#nfr-11), [NFR-14](03-architecture.md#nfr-14), [UP-4](03-architecture.md#up-4), [UP-32](03-architecture.md#up-32).

- [ ] <a id="m0-t1"></a>M0-T1 Repo, NestJS skeleton, lint, CI, `/healthz` (S)
  - [ ] M0-T1.1 Module layout: ingest, rules, alerts, devices, auth, ai, jobs, summaries
  - [ ] M0-T1.2 Config module with schema validation (fail fast on missing env)
  Acceptance: CI green; `/healthz` returns 200 without touching DB or Redis.
- [ ] <a id="m0-t2"></a>M0-T2 Deploy to Render free, plus external pinger (S)
  - [ ] M0-T2.1 `render.yaml`, env vars, TLS check from a laptop
  - [ ] M0-T2.2 Pinger every 10 min on `/healthz`; record real spin-up time
  Acceptance: service stays awake 24 h with the pinger; spin-up time recorded.
- [ ] <a id="m0-t3"></a>M0-T3 Repository layer with Drizzle, dual-dialect CI matrix (M)
  - [ ] M0-T3.1 sqlite/turso and postgres schema files kept in sync by a test
  - [ ] M0-T3.2 Migration runner and rollback procedure
  Acceptance: same repository test suite passes on libSQL and Postgres.
- [ ] <a id="m0-t4"></a>M0-T4 Spike S2: BullMQ on Render Key Value free (S)
  - [ ] M0-T4.1 Confirm `maxmemory-policy` can be set to `noeviction`; confirm size limit
  - [ ] M0-T4.2 Measure idle command rate and memory with one queue and two schedulers
  - [ ] M0-T4.3 If it fails, try Redis Cloud free and record latency
  Acceptance: documented go or no-go for each host.
- [ ] <a id="m0-t5"></a>M0-T5 Spike S1: Turso write accounting (S)
  - [ ] M0-T5.1 Insert 10,000 rows into an indexed table, read usage from the Turso dashboard or API
  - [ ] M0-T5.2 Delete 10,000 rows; check if deletes count
  Acceptance: measured writes per reading and per delete replace the estimates in [Estimated running cost and capacity](02-tech-stack.md#estimated-running-cost-and-capacity).
- [ ] <a id="m0-t6"></a>M0-T6 Spike S3: simulator and device scenarios (M)
  - [ ] M0-T6.1 Scenarios: normal, ammonia ramp, heat, fault (NaN, stuck), offline, burst, replay
  Acceptance: one command runs a scenario against any base URL ([FR-29](03-architecture.md#fr-29)).
- [ ] <a id="m0-t7"></a>M0-T7 Spike S4: firmware contract workshop (S)
  - [ ] M0-T7.1 Agree batching, boot_id, relative ages, raw ADC, retry rule, TLS approach, ADC scaling
  Acceptance: ingest contract ([API and integration contracts](03-architecture.md#api-and-integration-contracts)) signed off by the hardware lead ([Q-6](05-assumptions-risks-questions.md#q-6), [Q-7](05-assumptions-risks-questions.md#q-7)).
- [ ] <a id="m0-t8"></a>M0-T8 Spike S5: token sizes (S)
  - [ ] M0-T8.1 Build 5 real prompts, count tokens, estimate explanations per day under 100K tokens
  Acceptance: [DR-6](02-tech-stack.md#dr-6) numbers replaced by measurements.

### Milestone M1: Device registry and durable ingest
<a id="m1"></a>
Goal: readings from a device reach the DB safely.
Definition of done: simulator runs 1 hour at 40 devices with zero loss and zero duplicates; [A-M1-1](#a-m1-1) passes.
Depends on: M0.
Covers: [FR-1](03-architecture.md#fr-1) to [FR-5](03-architecture.md#fr-5), [FR-26](03-architecture.md#fr-26), [NFR-1](03-architecture.md#nfr-1) to [NFR-3](03-architecture.md#nfr-3), [UP-6](03-architecture.md#up-6) to [UP-11](03-architecture.md#up-11), [UP-35](03-architecture.md#up-35).

- [ ] <a id="m1-t1"></a>M1-T1 Device registry and key lifecycle (M)
  - [ ] M1-T1.1 Create, rotate, revoke endpoints (admin guarded later by M3)
  - [ ] M1-T1.2 Key generation, hashing, constant-time verify, per-device rate limit
  Acceptance: revoked key rejected on next request; key never logged.
- [ ] <a id="m1-t2"></a>M1-T2 Ingest endpoint and validation (M)
  - [ ] M1-T2.1 Strict schema, 16 KB cap, batch max 60
  - [ ] M1-T2.2 `captured_at` from `age_s`, `time_quality`, reject impossible ages
  - [ ] M1-T2.3 Response with `server_time_ms` and `config_version`
  Acceptance: contract tests from the golden payloads pass.
- [ ] <a id="m1-t3"></a>M1-T3 Idempotent storage (M)
  - [ ] M1-T3.1 Unique (device_id, boot_id, seq); batch insert in one transaction; count duplicates
  Acceptance: replay test stores each reading once; reboot with new boot_id works.
- [ ] <a id="m1-t4"></a>M1-T4 Cold-start behaviour (S)
  - [ ] M1-T4.1 Ingest returns 503 (not 200) if the DB write failed
  - [ ] M1-T4.2 Graceful shutdown drains in-flight requests on SIGTERM
  Acceptance: [A-M1-1](#a-m1-1) (kill -9 mid-batch loses no acknowledged reading).
- [ ] <a id="m1-t5"></a>M1-T5 NH3 calibration (M)
  - [ ] M1-T5.1 Calibration table and curve function; store raw and estimate
  Acceptance: changing a calibration changes new rows only; `calib_version` recorded.
- [ ] <a id="m1-t6"></a>M1-T6 `last_seen` throttling (S)
  - [ ] M1-T6.1 Write at most once per minute per device; first reading after offline always writes
  Acceptance: write meter shows at most 1 `devices` update per device-minute.
- [ ] <a id="m1-t7"></a>M1-T7 Write meter and degraded mode (M)
  - [ ] M1-T7.1 `usage_meter` increments; warnings at 70% and degraded mode at 90%
  Acceptance: forced low quota test switches to degraded mode and back.

### Milestone M2: Fault detection, rules and alert lifecycle
<a id="m2"></a>
Goal: correct, non-noisy alerts without AI.
Definition of done: scenario suite (ramp, spike, hover, fault, all-night breach) yields the expected alert rows.
Depends on: M1.
Covers: [FR-6](03-architecture.md#fr-6) to [FR-8](03-architecture.md#fr-8), [NFR-4](03-architecture.md#nfr-4), [NFR-14](03-architecture.md#nfr-14), [UP-12](03-architecture.md#up-12) to [UP-17](03-architecture.md#up-17), [UP-31](03-architecture.md#up-31).

- [ ] <a id="m2-t1"></a>M2-T1 Fault rules (M)
  - [ ] M2-T1.1 NaN or `dht_ok=false`, impossible range, stuck value over N readings, jump over limit, ADC saturation
  - [ ] M2-T1.2 One `SENSOR_FAULT` per episode, `FAULT_CLEARED` after M good readings
  Acceptance: faulty rows excluded from rules; fault event not repeated.
- [ ] <a id="m2-t2"></a>M2-T2 Threshold profiles and age bands (M)
  - [ ] M2-T2.1 Profile schema, default broiler profile, fallback profile
  - [ ] M2-T2.2 MQ135 warm-up flag and `out_of_spec` handling
  Acceptance: table-driven tests for each sensor and age band.
- [ ] <a id="m2-t3"></a>M2-T3 Rule engine with debounce (M)
  - [ ] M2-T3.1 Pure function over last N non-faulty readings; evaluates in `captured_at` order
  Acceptance: 2 breaching readings do not alert; 3 do.
- [ ] <a id="m2-t4"></a>M2-T4 Alert lifecycle (M)
  - [ ] M2-T4.1 Partial unique index for one non-resolved alert per (device, sensor)
  - [ ] M2-T4.2 Escalation, hysteresis (6 normal readings), ack, cooldown
  - [ ] M2-T4.3 `alert_events` history
  Acceptance: 24 h sustained breach yields 1 alert plus at most 1 escalation.
- [ ] <a id="m2-t5"></a>M2-T5 Template explanations (S)
  - [ ] M2-T5.1 Per sensor and level text with the measured value and generic safe actions
  Acceptance: every new alert has `explanation_source=template` before any LLM call.

### Milestone M3: Auth and read APIs
<a id="m3"></a>
Goal: secure, farm-scoped data access.
Definition of done: two-farm authorisation suite green on every endpoint.
Depends on: M1 (data), M2 (alerts).
Covers: [FR-13](03-architecture.md#fr-13), [FR-14](03-architecture.md#fr-14), [FR-21](03-architecture.md#fr-21), [FR-23](03-architecture.md#fr-23) (partial), [FR-25](03-architecture.md#fr-25), [FR-28](03-architecture.md#fr-28), [UP-28](03-architecture.md#up-28), [UP-29](03-architecture.md#up-29).

- [ ] <a id="m3-t1"></a>M3-T1 Users, roles, farms, pens (M)
  - [ ] M3-T1.1 Schema, seed admin, argon2id hashing
  Acceptance: admin can create a farm, pen and owner account.
- [ ] <a id="m3-t2"></a>M3-T2 Login, refresh, guards (M)
  - [ ] M3-T2.1 Short-lived access token, refresh rotation, logout
  Acceptance: expired token rejected; refresh works once.
- [ ] <a id="m3-t3"></a>M3-T3 Farm scoping in the repository layer (M)
  - [ ] M3-T3.1 Every query takes a scope object; lint rule blocks unscoped access
  Acceptance: cross-farm tests return 404 for every endpoint.
- [ ] <a id="m3-t4"></a>M3-T4 Read endpoints (M)
  - [ ] M3-T4.1 devices, latest, history with cursor, alerts, ack
  Acceptance: pagination stable under concurrent inserts.
- [ ] <a id="m3-t5"></a>M3-T5 OpenAPI and audit log (S)
  - [ ] M3-T5.1 Spec generated in CI; admin actions logged
  Acceptance: CI fails on spec drift.

### Milestone M4: Realtime
<a id="m4"></a>
Goal: dashboard updates without polling.
Definition of done: p95 reading-to-client latency <= 2 s on the simulator.
Depends on: M3.
Covers: [FR-15](03-architecture.md#fr-15), [UP-29](03-architecture.md#up-29), [UP-28](03-architecture.md#up-28).

- [ ] <a id="m4-t1"></a>M4-T1 socket.io gateway on the same server (M)
  - [ ] M4-T1.1 JWT handshake, rooms per farm, throttle `reading` to 1 per device per 5 s
  Acceptance: client gets events only for its farm.
- [ ] <a id="m4-t2"></a>M4-T2 Expiry and reconnect handling (S)
  - [ ] M4-T2.1 Disconnect on token expiry; client refresh and rejoin; missed-event catch-up via REST `since` parameter
  Acceptance: reconnect after 2 min loses no alert (client fetches since last id).
- [ ] <a id="m4-t3"></a>M4-T3 Wake-on-connect check (S)
  - [ ] M4-T3.1 Verify a new WebSocket connection wakes a sleeping Render service and the client shows a "waking up" state
  Acceptance: documented behaviour and measured wake time.

### Milestone M5: Scheduler, offline detection and reconciler
<a id="m5"></a>
Goal: background work that survives sleep, restart and Redis loss.
Definition of done: [A-M5-1](#a-m5-1) to [A-M5-3](#a-m5-3) pass.
Depends on: [M0-T4](#m0-t4), M2.
Covers: [FR-12](03-architecture.md#fr-12), [FR-24](03-architecture.md#fr-24), [UP-1](03-architecture.md#up-1) to [UP-5](03-architecture.md#up-5), [UP-23](03-architecture.md#up-23).

- [ ] <a id="m5-t1"></a>M5-T1 Queue module (M)
  - [ ] M5-T1.1 BullMQ queues, retry with backoff, `removeOnComplete` and `removeOnFail` caps, graceful close
  - [ ] M5-T1.2 Startup check of eviction policy; fail loudly
  Acceptance: queue memory stays flat over a 24 h run.
- [ ] <a id="m5-t2"></a>M5-T2 Offline sweep (M)
  - [ ] M5-T2.1 Scheduler every 2 min; 10 min threshold; 3 min boot grace
  - [ ] M5-T2.2 Recovery on first reading; `device_events` and WebSocket push
  Acceptance: [A-M5-1](#a-m5-1) simulator stop yields OFFLINE within 12 min; restart yields ONLINE.
- [ ] <a id="m5-t3"></a>M5-T3 Boot catch-up and keep-alive (S)
  - [ ] M5-T3.1 On boot run sweep and aggregate catch-up; document the pinger and its limits
  Acceptance: [A-M5-2](#a-m5-2) stop service 30 min with devices silent; on wake, OFFLINE events appear within 5 min.
- [ ] <a id="m5-t4"></a>M5-T4 Idempotent schedulers and reconciler (M)
  - [ ] M5-T4.1 `upsertJobScheduler` at boot; deterministic job ids
  - [ ] M5-T4.2 Reconciler re-enqueues pending explanations older than 2 min
  Acceptance: [A-M5-3](#a-m5-3) flush Redis mid-run; all pending alerts still get explained and schedulers resume.

### Milestone M6: AI explanations
<a id="m6"></a>
Goal: useful explanations that never block or break alerts.
Definition of done: 20-scenario evaluation passes; behaviour under 429 verified.
Depends on: M2, M5.
Covers: [FR-9](03-architecture.md#fr-9) to [FR-11](03-architecture.md#fr-11), [FR-20](03-architecture.md#fr-20), [NFR-7](03-architecture.md#nfr-7), [NFR-8](03-architecture.md#nfr-8), [UP-18](03-architecture.md#up-18) to [UP-20](03-architecture.md#up-20), [UP-22](03-architecture.md#up-22).

- [ ] <a id="m6-t1"></a>M6-T1 KB schema, seed and tag lookup (M)
  - [ ] M6-T1.1 Seed entries: per sensor and level, plus combinations (high NH3 with high humidity, heat with high humidity, gas spike with normal NH3)
  - [ ] M6-T1.2 Lookup ranks by specificity; top 3 entries
  Acceptance: combined condition returns the combined entry first.
- [ ] <a id="m6-t2"></a>M6-T2 Prompt builder (S)
  - [ ] M6-T2.1 Numbers, profile, KB text only; JSON-only output; hard `max_tokens`
  Acceptance: snapshot test shows no names or contacts in prompts.
- [ ] <a id="m6-t3"></a>M6-T3 Budget router and throttle (M)
  - [ ] M6-T3.1 `llm_usage_daily`; 70B then 8B then template; 1 call per (device, sensor, level) per 30 min
  Acceptance: forced exhausted budget leaves alerts with templates and no errors.
- [ ] <a id="m6-t4"></a>M6-T4 Worker with retry (M)
  - [ ] M6-T4.1 429 and 5xx backoff; model ids from config; update alert and push `alert` event
  Acceptance: Groq mock returning 429 for 10 min recovers afterwards.
- [ ] <a id="m6-t5"></a>M6-T5 Output safety (S)
  - [ ] M6-T5.1 Schema validation, denylist (drug names and doses), fallback to template
  Acceptance: injected unsafe output is never stored.
- [ ] <a id="m6-t6"></a>M6-T6 Language handling (S)
  - [ ] M6-T6.1 `lang` allow list; English fallback; mark non-English as beta until reviewed
  Acceptance: unsupported value returns 400 with the allowed list.

### Milestone M7: Aggregates, summaries and retention
<a id="m7"></a>
Goal: trustworthy history and narratives.
Definition of done: aggregates reproducible from raw data; retention runs without affecting ingest.
Depends on: M5.
Covers: [FR-16](03-architecture.md#fr-16) to [FR-18](03-architecture.md#fr-18), [FR-22](03-architecture.md#fr-22), [NFR-12](03-architecture.md#nfr-12), [NFR-13](03-architecture.md#nfr-13), [UP-23](03-architecture.md#up-23) to [UP-27](03-architecture.md#up-27).

- [ ] <a id="m7-t1"></a>M7-T1 Period aggregates (M)
  - [ ] M7-T1.1 Farm-zone boundaries, night attribution, unique (device, date, period)
  Acceptance: boundary tests at 05:59, 06:00, 21:59, 22:00 and across midnight.
- [ ] <a id="m7-t2"></a>M7-T2 Daily summary (M)
  - [ ] M7-T2.1 Weighted by sample count; `coverage_pct`
  Acceptance: a day with a 6-hour outage reports 75% coverage and correct averages.
- [ ] <a id="m7-t3"></a>M7-T3 Narratives (M)
  - [ ] M7-T3.1 Daily narrative from last 3 to 7 summaries using the 8B model; monthly and yearly later
  Acceptance: narrative failure leaves the numeric summary intact.
- [ ] <a id="m7-t4"></a>M7-T4 Catch-up recompute (S)
  - [ ] M7-T4.1 Job recomputes missing periods for the last 7 days on boot and daily
  Acceptance: deleting an aggregate row is repaired within one run.
- [ ] <a id="m7-t5"></a>M7-T5 Retention (S)
  - [ ] M7-T5.1 Batched deletes (for example 5,000 rows per run), off-peak, counted in the write meter
  Acceptance: ingest p95 unchanged during a retention run.

### Milestone M8: AI chat
<a id="m8"></a>
Goal: safe follow-up Q&A.
Depends on: M6, M3.
Covers: [FR-19](03-architecture.md#fr-19), [UP-21](03-architecture.md#up-21).

- [ ] <a id="m8-t1"></a>M8-T1 Chat endpoint (M)
  - [ ] M8-T1.1 Body validation; last 3 to 5 turns; per-user daily cap
  Acceptance: cap returns a clear message, not an error page.
- [ ] <a id="m8-t2"></a>M8-T2 Guardrails (M)
  - [ ] M8-T2.1 System prompt scope; no tools; farm-only context; injection test set
  Acceptance: 20 injection attempts leak nothing and stay on topic.

### Milestone M9: Hardening, docs and operations
<a id="m9"></a>
Goal: handover-ready backend.
Depends on: M1 to M8.
Covers: [FR-24](03-architecture.md#fr-24), [FR-28](03-architecture.md#fr-28), [NFR-6](03-architecture.md#nfr-6), [NFR-10](03-architecture.md#nfr-10), [UP-6](03-architecture.md#up-6), [UP-11](03-architecture.md#up-11), [UP-30](03-architecture.md#up-30), [UP-32](03-architecture.md#up-32) to [UP-34](03-architecture.md#up-34).

- [ ] <a id="m9-t1"></a>M9-T1 Secrets and rotation runbook (S)
- [ ] <a id="m9-t2"></a>M9-T2 Admin safety: soft delete, confirmation by name, in-use profile guard (S)
- [ ] <a id="m9-t3"></a>M9-T3 Metrics endpoint and quota dashboards (S)
- [ ] <a id="m9-t4"></a>M9-T4 Backups, restore drill and runbook (M)
  Acceptance: a restore drill rebuilds the DB from the latest export on a clean instance.
- [ ] <a id="m9-t5"></a>M9-T5 API docs and integration guide for frontend and firmware teams (M)
  Acceptance: a new frontend developer completes login to live chart using the docs only.
- [ ] <a id="m9-t6"></a>M9-T6 Load and chaos run, results recorded (M)

### Milestone M10: Go-live migration (conditional on Q-1)
<a id="m10"></a>
Goal: move off free tiers without data loss.
Depends on: decision on host.
Covers: [UP-2](03-architecture.md#up-2), [UP-34](03-architecture.md#up-34).

- [ ] <a id="m10-t1"></a>M10-T1 Provision always-on web, Postgres and Redis; configure `noeviction` (M)
- [ ] <a id="m10-t2"></a>M10-T2 Data migration from Turso to Postgres with count and checksum verification (M)
- [ ] <a id="m10-t3"></a>M10-T3 Cutover plan: firmware base URL via DNS name from day one so no device reflash is needed (S)
  Acceptance: dry run on staging with a copy of production data.
- [ ] <a id="m10-t4"></a>M10-T4 Decide whether to split web and worker services (S)

### Acceptance scenarios referenced above

- **<a id="a-m1-1"></a>A-M1-1** Send batches continuously, `kill -9` the process mid-batch, restart; every reading that received 200 is present exactly once.
- **<a id="a-m5-1"></a>A-M5-1** Stop a simulated device; OFFLINE within 12 min while the service is awake; ONLINE on the next reading.
- **<a id="a-m5-2"></a>A-M5-2** Stop the service for 30 min with all devices silent; after wake, OFFLINE events within 5 min and no false OFFLINE for devices that resumed within the 3-min grace.
- **<a id="a-m5-3"></a>A-M5-3** Flush Redis while explanations are pending; all pending alerts are explained and schedulers resume.

Next: [Assumptions, Risks, Open Questions](05-assumptions-risks-questions.md)
