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

## Team Roles & Milestones

Work is split by role so each person can work independently and converge at defined gates. Full task lists and acceptance criteria are in [`docs/prd/04-requirements.md`](docs/prd/04-requirements.md).

| Role | Milestones | Branch |
|---|---|---|
| **Service Design (SD)** | SD1 Foundations, spikes and ports | `milestone/sd1-foundations-ports` |
| | SD2 Device registry and durable ingest | `milestone/sd2-device-registry-ingest` |
| | SD3 Fault detection, rules and alert lifecycle | `milestone/sd3-faults-rules-alerts` |
| | SD4 Auth, farm scoping and read services | `milestone/sd4-auth-read-services` |
| | SD5 Realtime | `milestone/sd5-realtime` |
| | SD6 Scheduler, offline detection and reconciler | `milestone/sd6-scheduler-offline-reconciler` |
| | SD7 Aggregates, retention and instrumentation | `milestone/sd7-aggregates-retention-instrumentation` |
| **API Planning & Integration (API)** | API1 Endpoint planning and contract | `milestone/api1-endpoint-planning-contract` |
| | API2 Frontend integration | `milestone/api2-frontend-integration` |
| | API3 Hardware integration | `milestone/api3-hardware-integration` |
| | API4 Documentation and handover | `milestone/api4-docs-handover` |
| **AI Integration (AI)** | AI1 AI workflow and knowledge base | `milestone/ai1-workflow-knowledge-base` |
| | AI2 Alert explanation engine | `milestone/ai2-explanation-engine` |
| | AI3 Narratives and chat | `milestone/ai3-narratives-chat` |
| **Validation & Testing (QA)** | QA1 Test harness and endpoint verification | `milestone/qa1-test-harness-endpoint-verification` |
| | QA2 Subsystem verification | `milestone/qa2-subsystem-verification` |
| | QA3 Full integration test and sign-off | `milestone/qa3-full-integration-test` |
| **Monitoring & Deployment (OPS)** | OPS1 Backend monitoring | `milestone/ops1-backend-monitoring` |
| | OPS2 Frontend monitoring | `milestone/ops2-frontend-monitoring` |
| | OPS3 Operations readiness | `milestone/ops3-operations-readiness` |
| | OPS4 Go-live and post-launch (conditional on Q-1) | `milestone/ops4-go-live` |

Modules talk to each other only through shared interfaces (ports) with stub implementations, so unfinished work never blocks anyone. See [Ports](docs/prd/04-requirements.md#ports-how-sd-and-ai-work-independently).

### Convergence gates

| Gate | Condition | Unblocks |
|---|---|---|
| G0 Contracts frozen | API1 sign-off and SD1 ports published | SD2, SD4, AI1, AI2 |
| G1 First vertical slice | SD1 + SD2 + SD3 with simulator | QA1, API3 |
| G2 Frontend-ready | SD4 + SD5 done, QA1 verification passed, API2 shapes agreed | Frontend integration |
| G3 AI plugged in | AI2 passes evaluation, SD3 and SD6 exist | AI3, real AI engine replaces stubs |
| G4 Feature complete | SD, AI, API1 to API3 done | QA3, OPS3, OPS4 |
| G5 Release | QA3 sign-off, OPS1 to OPS3 done, Q-1 decided | OPS4 go-live |

---

## Branching & Merge Workflow

`main` is the reconciliation branch. It must always build, boot (with stubs for unfinished ports) and pass tests.

1. **One long-lived branch per milestone**, named `milestone/<id>-<name>` (table above). Start from `main`; the role that owns the milestone works on it.
2. **Optional task branches** for parallel work inside a milestone: `task/<task-id>-<name>` (for example `task/sd3-t4-alert-lifecycle`), merged into the milestone branch by pull request.
3. **Sync often.** Merge `main` into your milestone branch at least daily, and before you start a new task, so conflicts stay small.
4. **Merge to `main` through a pull request**, when a milestone (or a self-contained part of one) is done or a gate needs it. Another role reviews; the CI must pass. Do not merge a milestone that breaks the build or boots only with real services.
5. **Contract changes go first.** A change to a shared interface, DTO or event type in `contracts/` is its own small pull request into `main`, reviewed by every role that consumes it. Then implementation branches merge `main` and continue.
6. **Who edits what.** SD owns core module code. API may make small changes to controllers, DTOs and response shapes in SD modules (SD reviews). AI owns the `ai` and `knowledge` modules. QA and OPS change tests and monitoring configuration, not product code; defects go back to the owning role.
7. **Database migrations:** one migration per pull request. If two branches add one, regenerate yours after merging `main`.
8. Delete task branches after merging. Milestone branches stay until the milestone is closed.

---

## Documentation

Full PRD specifications are located in the [`docs/prd/`](docs/prd/index.md) directory:

- [**01. Overview**](docs/prd/01-overview.md) — Problem statement, target users, goals, non-goals, and success metrics.
- [**02. Tech Stack & Tools**](docs/prd/02-tech-stack.md) — Tool choices, free tier capacity calculations, and decision records (DR-1 to DR-8).
- [**03. Architecture**](docs/prd/03-architecture.md) — Functional/non-functional requirements, data schemas, API contracts, and failure handling.
- [**04. Requirements & Milestones**](docs/prd/04-requirements.md) — Roles, ports, convergence gates, and task breakdowns by milestone (SD1–SD7, API1–API4, AI1–AI3, QA1–QA3, OPS1–OPS4) with acceptance criteria.
- [**05. Assumptions, Risks & Questions**](docs/prd/05-assumptions-risks-questions.md) — Assumption ledger, risk register, and open technical questions.
