# Overview

Status: Draft | Owner: Prosper (Backend & AI Team, SARE) | Last updated: 2026-10-16 | Version: 0.1

Scope: the backend and AI layer of the Air Quality Monitoring System (AQMS), as defined in SARE's AQMS Project Requirement Document (sections 6 and 9) and the earlier "AQMS Backend Architecture Plan". Frontend, firmware, PCB and casing are out of scope except where this backend depends on them.

How to read this: every fact about the outside world carries a status in the [Assumption Ledger](05-assumptions-risks-questions.md#assumption-ledger): Verified, Contradicted, or Unverified. Files most likely to change: [02-tech-stack.md](02-tech-stack.md) (hosting decisions at go-live), [03-architecture.md](03-architecture.md) (threshold values pending SARE's poultry advisor, ingest contract pending the firmware workshop), and [open questions](05-assumptions-risks-questions.md#open-questions).

## Problem and opportunity

Poultry pens build up ammonia, heat and moisture, and farmers often notice only after birds are affected. Guidance cited by the Ontario ag ministry puts the ammonia maximum at 25 ppm, with lung surface damage above 10 ppm and more respiratory problems above 20 ppm; the EU welfare directive uses 20 ppm as a threshold; recommended relative humidity is roughly 50 to 70%, and wet litter drives ammonia up (sources in [Sources](05-assumptions-risks-questions.md#sources)).

AQMS is a wall-mounted ESP8266 device (DHT11 temperature and humidity, MQ135 gas sensor) that posts readings to a backend. The backend must store them, judge them against safe ranges, raise alerts, explain alerts in plain language (AI), summarize trends, and feed a real-time dashboard. This PRD specifies that backend.

## Target audience

| Segment | Pain | Current workaround | Evidence |
|---|---|---|---|
| Farm owner or pen operator (end user of the dashboard) | Cannot see air conditions continuously; ammonia is hard to judge by smell | Walk the pen; smell | Ammonia is typically not noticed by smell until 50 to 60 ppm (UGA broiler tip). Farmer interviews not done: Unverified |
| SARE team (builders, demo audience) | Needs a working end-to-end system on a zero-budget stack | Free tiers | SARE PRD; stack choice made by the user on 2026-10-16 |
| SARE admin (provisioning, support) | Needs to create devices, keys, farms, thresholds without touching the database | None yet | Derived from PRD section 6.2 |

## Market and competitors

Not applicable to this document. Product positioning belongs to SARE's product PRD. This PRD only defines the backend.

## Positioning and ship plan

- **MVP (v1):** device ingest with authentication, storage, fault detection, threshold alerts with a proper lifecycle, offline detection, AI explanation with graceful fallback, period and daily summaries, REST plus WebSocket for the dashboard.
- **Dev profile (now):** all free tiers: Render free web service, Turso free, a free Redis for BullMQ, Groq free. Cost $0. Accepted limits are in [Tech Stack and Tools](02-tech-stack.md#estimated-running-cost-and-capacity).
- **Go-live profile (later):** hosting target to be chosen ([Q-1](05-assumptions-risks-questions.md#q-1)). The code is built to move: repository layer over the DB, Redis treated as disposable, no host-specific features.
- **First vertical slice:** simulator -> ingest -> store -> rule -> alert row -> `GET /devices/latest`. This proves the riskiest path before AI and summaries.

## Groundwork (settle before or during M0)

- Accounts: Render, Turso, a Redis host, Groq, GitHub (repo and CI). Owner: backend team.
- **Firmware contract workshop** with the hardware team (batching, retry rules, boot id and sequence number, time sync, TLS approach, raw ADC). Without it the ingest contract is a guess. See [Q-6](05-assumptions-risks-questions.md#q-6), [Q-7](05-assumptions-risks-questions.md#q-7).
- **Threshold source:** SARE needs a named poultry advisor to confirm alert thresholds and whether they vary by bird age ([Q-3](05-assumptions-risks-questions.md#q-3)). The values in the source architecture doc do not match published guidance ([thresholds](03-architecture.md#data-model-sketch)).
- **Sensor reality check:** MQ135 is not an ammonia sensor in the strict sense. The datasheet lists NH3 10 to 300 ppm among several cross-sensitive gases, needs 24 hours or more of preheat and requires calibration. PPM numbers will be estimates; the product must label them so ([Q-8](05-assumptions-risks-questions.md#q-8)).
- **Data protection:** farm user accounts hold personal data. Whether Nigerian data protection law imposes duties on SARE here has not been checked ([Q-11](05-assumptions-risks-questions.md#q-11)).

## Goals, non-goals, success metrics

**Goals**
- G1: No accepted reading is lost to a backend restart.
- G2: A breach shows on the dashboard within seconds, with a useful explanation soon after, and never blocks on the AI provider.
- G3: Offline and faulty devices are surfaced, not silently shown as "normal".
- G4: The whole system runs at $0 in dev and can move to a paid host without a rewrite.

**Non-goals (v1)**
- No firmware, frontend, PCB or casing work.
- No actuator control (fans, heaters) even though SARE's overview lists "Automation". Parked as [Q-13](05-assumptions-risks-questions.md#q-13).
- No SMS, WhatsApp or push delivery in v1; a channel interface exists, channels are [Q-4](05-assumptions-risks-questions.md#q-4).
- No multi-region, no high availability, no SLA on the free profile.
- No ML model training; AI means LLM explanation and narrative only.

| Metric | Baseline | Target | Timeframe | How measured |
|---|---|---|---|---|
| Reading durability | none | 100% of readings acknowledged with 2xx are in the DB after a forced kill | M1 done | kill-9 test ([A-M1-1](04-requirements.md#a-m1-1)) |
| Reading to dashboard push latency (p95, service awake) | none | <= 2 s | M4 done | simulator timestamps vs WS receive |
| Breach to alert row (p95) | none | <= 1 s after commit | M2 done | integration test |
| Explanation availability | none | 100% of alerts show a template explanation immediately; LLM text for >= 90% within 60 s while within budget | M6 done | alert table, `explanation_source` |
| Offline detection (service awake) | none | <= 12 min after last reading (10 min threshold + 2 min sweep) | M5 done | simulator stop test |
| Free-tier headroom | none | Turso writes <= 80% of 10M/month at pilot device count; LLM tokens <= 80% of daily free limit | continuous | write meter, `llm_usage_daily` |

Primary metric: durability plus alert latency (this is a safety-adjacent product). Guardrails: false-offline alerts per device per week <= 1 (while the service is awake); LLM-caused alert delay = 0 (AI is never on the alert path).

Next: [Tech Stack and Tools](02-tech-stack.md)
