# AQMS Backend

Backend service and AI intelligence layer for the **Air Quality Monitoring System (AQMS)**, an IoT monitoring platform designed for poultry pens. The system ingests environmental data (temperature, humidity, ammonia/gas) from ESP8266 hardware, identifies sensor faults, evaluates threshold rules, alerts farm operators in real time, and provides context-aware AI explanations and daily summaries.

---

## Key Features

- **Durable, Idempotent Ingest (`/api/v1/ingest`):** Accepts batches of 1–60 readings; authoritative server-calculated timestamps via relative reading ages; de-duplication via `(device_id, boot_id, seq)`.
- **Fault & Threshold Rule Engine:** Pure deterministic evaluation over the last *N* readings. Distinguishes sensor faults (saturation, stuck values, impossible jumps, DHT failures) from environmental breaches. Supports age-based temperature curves and hysteresis to eliminate alert flapping.
- **Decoupled AI Layer (Groq):** Alerts immediately emit safe, localized template advice, enqueuing asynchronous Groq LLM jobs (`llama-3.3-70b-versatile` for explanations, `llama-3.1-8b-instant` for daily narratives). AI is never on the critical path of alert generation.
- **Disposable Queue Architecture:** Database is the sole source of truth; Redis (BullMQ) is completely disposable. A reconciler restores pending jobs on startup.
- **Offline Device Sweeper:** Background jobs check for silent devices (>10 min) with boot-grace handling, surfacing `DEVICE_OFFLINE` / `DEVICE_ONLINE` events.
- **Multi-Tenant Farm Scoping:** Repository-enforced data boundaries, hashed device API keys, argon2id password hashing, and JWT-authenticated Socket.io rooms.
- **Dual-Dialect Database Portability:** Repository pattern with Drizzle ORM supporting Turso (libSQL) for dev ($0 cost profile) and PostgreSQL for production.

---

## Tech Stack

| Layer | Technology | Role & Notes |
|---|---|---|
| **Runtime & Framework** | Node.js, NestJS, TypeScript | Modular architecture, DI, validation pipes |
| **Database & ORM** | Turso (libSQL) & Drizzle ORM | Zero-cost dev database; portable to Postgres |
| **Queues & Scheduling** | BullMQ & Redis (Render Key Value) | Async LLM jobs, offline sweep, aggregations |
| **LLM Provider** | Groq (`llama-3.3-70b`, `llama-3.1-8b`) | Token-budgeted explanations & narratives |
| **Transports** | REST (`/api/v1`), Socket.io | Dashboard queries and low-latency event push |
| **Dev Hosting** | Render Free Web Service | Single-process free tier with external pinger |

---

## System Architecture

```
[ ESP8266 Device ]
  │  (HTTPS Batch POST with device key)
  ▼
[ Ingest API ] ───► [ Turso DB (Source of Truth) ]
  │
  ├─► [ Fault & Threshold Rules ] ──► [ Alert Created (Template Advice) ]
  │                                           │ (Socket.io push)
  │                                           ▼
  │                                  [ BullMQ Job Enqueued ]
  │                                           │
  │                                           ▼
  │                                  [ Groq LLM Worker ] ──► Update Alert
  ▼
[ Dashboard Client ] ◄── (REST & Socket.io) ── [ Auth & Read APIs ]
```

---

## Ingest Contract Snapshot

Devices authenticate with a 256-bit hashed key and submit batches using relative age offsets (`age_s`):

```http
POST /api/v1/ingest
x-device-key: <device-api-key>
Content-Type: application/json

{
  "boot_id": "a1b2c3",
  "seq": 120,
  "fw": "0.3.1",
  "readings": [
    { "age_s": 50, "t": 28.6, "h": 68.0, "gas_raw": 412, "dht_ok": true },
    { "age_s": 40, "t": 28.7, "h": 68.0, "gas_raw": 415, "dht_ok": true }
  ]
}
```

**Response (`200 OK`):**
```json
{
  "accepted": 2,
  "duplicate": 0,
  "rejected": [],
  "server_time_ms": 1760625600000,
  "config_version": 3
}
```

---

## Roadmap & Milestones

The project is structured into sequential milestones to prove stability before adding complexity:

- **M0: Foundations & Spikes** — Project skeleton, dual-dialect Drizzle setup, Render deploy, and capacity spikes.
- **M1: Device Registry & Durable Ingest** — Device API keys, batch validation, idempotent storage, cold-start safety.
- **M2: Fault Detection & Alert Lifecycle** — Sensor fault rules, threshold engine, debounce, hysteresis, and template explanations. *(M0–M2 constitute the first vertical slice)*
- **M3: Auth & Read APIs** — User sessions, RBAC, farm scoping, history and device status endpoints.
- **M4: Realtime** — Socket.io gateway with farm-scoped event rooms.
- **M5: Schedulers & Offline Sweeper** — BullMQ queues, offline detection, and recovery reconciler.
- **M6: AI Explanations** — Tagged knowledge base, token budget router, and Groq worker integration.
- **M7: Aggregates, Summaries & Retention** — Period/daily summaries and raw reading data pruning.
- **M8: AI Chat** — Farm-scoped contextual Q&A assistance.
- **M9: Hardening & Operational Runbooks** — Auditing, backup drills, and chaos/load testing.
- **M10: Go-Live Migration** — Seamless transition from free tier to dedicated PostgreSQL and always-on instances.

---

## Documentation

Full PRD specifications are located in the [`docs/prd/`](docs/prd/index.md) directory:

- [**01. Overview**](docs/prd/01-overview.md) — Problem statement, target users, goals, non-goals, and success metrics.
- [**02. Tech Stack & Tools**](docs/prd/02-tech-stack.md) — Tool choices, free tier capacity calculations, and decision records (DR-1 to DR-8).
- [**03. Architecture**](docs/prd/03-architecture.md) — Functional/non-functional requirements, data schemas, API contracts, and failure handling.
- [**04. Requirements & Milestones**](docs/prd/04-requirements.md) — Detailed task breakdowns (M0–M10) and acceptance criteria.
- [**05. Assumptions, Risks & Questions**](docs/prd/05-assumptions-risks-questions.md) — Assumption ledger, risk register, and open technical questions.
