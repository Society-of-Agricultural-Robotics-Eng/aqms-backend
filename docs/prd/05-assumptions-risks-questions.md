# Assumptions, Risks, Open Questions

## Assumption Ledger

Date checked: 2026-10-16 unless stated.

| ID | Assumption | Status | Evidence and source | 
|---|---|---|---|
| <a id="a-1"></a>A-1 | Render free web service sleeps after 15 min idle, wakes on HTTP or new WebSocket connection, 750 hours per workspace, may restart any time | Verified | Render free-tier docs |
| <a id="a-2"></a>A-2 | Render free Key Value does not persist to disk and may restart at any time | Verified | Render free-tier docs |
| <a id="a-3"></a>A-3 | Render free Key Value size is 25 MB | Unverified | Third-party deployment guides only |
| <a id="a-4"></a>A-4 | Render free Postgres expires after 30 days | Verified | Render free-tier docs |
| <a id="a-5"></a>A-5 | Turso free: 10M rows written and 500M rows read per month; writes beyond the quota fail with BLOCKED | Verified | Turso pricing page and billing docs |
| <a id="a-6"></a>A-6 | Turso counts index entries and deletes as written rows | Unverified | Docs say inserts and updates count; others not stated. Spike S1 |
| <a id="a-7"></a>A-7 | Groq `llama-3.3-70b-versatile` free: 30 RPM, 1K RPD, 12K TPM, 100K TPD | Verified | Groq rate-limit docs table, confirmed by secondary guides |
| <a id="a-8"></a>A-8 | Source doc claim "14,400 req/day for the 70B model" | Contradicted | 14,400 RPD is the 8B model; 70B is 1,000 RPD |
| <a id="a-9"></a>A-9 | Groq `llama-3.1-8b-instant` free: 14,400 RPD, 500K TPD, 6K TPM | Verified (secondary) | Secondary guides citing Groq docs |
| <a id="a-10"></a>A-10 | Groq does not retain inputs or outputs by default | Unverified | One secondary source |
| <a id="a-11"></a>A-11 | Upstash free (500K commands per month) is unsuitable for BullMQ | Verified limit; polling rate anecdotal | Upstash pricing; Fly community reports |
| <a id="a-12"></a>A-12 | Redis Cloud free: 30 MB, 30 connections, deleted after 14 days of inactivity | Verified | Redis docs and support |
| <a id="a-13"></a>A-13 | BullMQ requires `noeviction`; completed jobs are kept forever unless removal is configured | Verified | BullMQ docs |
| <a id="a-14"></a>A-14 | Source doc "Gemini embedding 1,500 req/day" and model name `text-embedding-001` | Contradicted or moot | Current models are `gemini-embedding-001` and `gemini-embedding-2`; free limits not published on the official page; embeddings dropped from design |
| <a id="a-15"></a>A-15 | Pinecone Starter inactivity pause | Unverified | Sources conflict (7 days vs 3 weeks); moot after [DR-5](02-tech-stack.md#dr-5) |
| <a id="a-16"></a>A-16 | Heroku student credit, Eco sleep, 24 h dyno cycling, Scheduler best-effort with 10-min minimum | Verified; not applicable | Heroku docs |
| <a id="a-17"></a>A-17 | Railway "~$5 per month all-in-one" | Contradicted for always-on API + Postgres + Redis | Railway pricing docs and cost analyses suggest $10 to $30 or more; moot |
| <a id="a-18"></a>A-18 | MQ135 NH3 range 10 to 300 ppm, 24 h+ preheat, cross-sensitive, needs calibration | Verified | Hanwei datasheet and vendor notes |
| <a id="a-19"></a>A-19 | DHT11: 0 to 50 °C ±2 °C, 20 to 90 % RH ±5 %, 1 Hz | Verified | Multiple vendor datasheets |
| <a id="a-20"></a>A-20 | Ammonia max 25 ppm; above 10 ppm lung damage; above 20 ppm respiratory issues; EU threshold 20 ppm; RH 50 to 70 % | Verified | Ontario ag factsheet, PMC study, UGA broiler tip |
| <a id="a-21"></a>A-21 | Broiler temperature falls from about 30 to 32 °C at placement to about 19 to 20 °C by day 42 | Verified | Ontario factsheet, PMC study |
| <a id="a-22"></a>A-22 | ESP8266 bare ADC accepts 0 to 1 V; dev boards differ | Verified | Forum and tutorial sources |
| <a id="a-23"></a>A-23 | ESP8266 TLS validation needs correct time; `setInsecure()` accepts any certificate; TLS handshake can take seconds | Verified | ESP8266 Arduino core examples and issue threads |
| <a id="a-24"></a>A-24 | Drizzle supports sqlite, turso and postgresql dialects | Verified | Drizzle release notes. Sharing one schema file across dialects: Unverified (assumed separate files) |
| <a id="a-25"></a>A-25 | pg-boss is maintained, uses SKIP LOCKED, has cron scheduling, Node 20+ | Verified | pg-boss repository and npm |
| <a id="a-26"></a>A-26 | Neon free: 100 CU-hours and 0.5 GB per project, scale to zero after 5 min | Verified | Neon pricing |
| <a id="a-27"></a>A-27 | About 1,500 tokens per explanation prompt and answer | Unverified assumption | Spike S5 |
| <a id="a-28"></a>A-28 | Pilot size is up to 40 devices at 10 s interval | Unverified | Taken from source doc; [Q-5](#q-5) |
| <a id="a-29"></a>A-29 | Turso free plan has 1-day point-in-time recovery | Unverified | Secondary sources only |

## Risk register

| ID | Risk | Likelihood | Impact | Mitigation | Owner |
|---|---|---|---|---|---|
| <a id="r-1"></a>R-1 | Free-tier limits (Turso writes, Groq tokens) hit sooner than estimated | High | Medium | Spikes S1 and S5, meters, degraded mode, template fallback | Backend |
| <a id="r-2"></a>R-2 | MQ135 NH3 estimates are inaccurate, causing false alarms or missed danger | High | High | Label as estimate, calibration workflow, warm-up handling, advisor review, consider a dedicated NH3 sensor | SARE hardware, advisor |
| <a id="r-3"></a>R-3 | Service sleeps while all devices are offline | High (dev) | Medium | Pinger, boot catch-up, always-on host at go-live | Backend |
| <a id="r-4"></a>R-4 | Firmware contract not agreed, ingest changes late | Medium | High | Workshop in M0, golden payloads, `fw` version gate | Backend, hardware |
| <a id="r-5"></a>R-5 | Thresholds not confirmed by a poultry expert | Medium | High | Configurable profiles, advisor sign-off before pilot | SARE lead |
| <a id="r-6"></a>R-6 | Dev-to-Postgres migration harder than expected | Medium | Medium | Dual-dialect CI from M0, portable schema | Backend |
| <a id="r-7"></a>R-7 | Redis free tier unusable with BullMQ | Medium | Medium | Spike S2; fallback Redis Cloud; last resort pg-boss | Backend |
| <a id="r-8"></a>R-8 | AI advice judged unsafe or wrong | Low to medium | High | Constrained prompts, denylist, human review sample, disclaimers | Backend |
| <a id="r-9"></a>R-9 | Free accounts suspended or terms change | Medium | Medium | Backups, runbook, Redis disposable | Backend |
| <a id="r-10"></a>R-10 | Account ownership: dev accounts are on a personal login | Medium | Medium | Move to a SARE-owned org account before pilot ([Q-12](#q-12)) | SARE lead |

## Open questions

| ID | Question | Why it matters | Owner / evidence path | Needed by |
|---|---|---|---|---|
| <a id="q-1"></a>Q-1 | Where does go-live run, and with what budget? | Decides DB, always-on instance, cost | SARE lead | before M10 |
| <a id="q-2"></a>Q-2 | Who logs in (farmers, SARE staff), how are accounts created, one farm or many? | Drives auth, scoping, data protection | SARE lead, frontend lead | before M3 |
| <a id="q-3"></a>Q-3 | Bird type and age: broilers or layers, and who signs off thresholds, including age bands? | Temperature bands, safety of alerts | Poultry advisor | before M2 |
| <a id="q-4"></a>Q-4 | Which alert channels beyond the dashboard (SMS, WhatsApp, email)? | Farmers may not watch a dashboard | SARE lead | v1.1 planning |
| <a id="q-5"></a>Q-5 | Real device count and sampling interval for the pilot? | Write budget, capacity | SARE lead, hardware | before M1 |
| <a id="q-6"></a>Q-6 | Can firmware batch readings, send `boot_id`, `seq`, relative ages, and buffer while offline? | Ingest contract and durability | Hardware lead | M0 |
| <a id="q-7"></a>Q-7 | How will firmware do TLS (pinned fingerprint, CA store with NTP, or `setInsecure`)? | Real security of device traffic | Hardware lead | M0 |
| <a id="q-8"></a>Q-8 | Will SARE label NH3 as an estimate, and will MQ135 stay or be replaced by an NH3-specific sensor? | Trust in the headline metric | SARE lead, hardware | before pilot |
| <a id="q-9"></a>Q-9 | Which languages must the AI support, and who can review them (for example Yoruba)? | LLM quality in local languages is unverified | SARE lead | before M6 |
| <a id="q-10"></a>Q-10 | Raw data retention: is 90 days right, and who needs history longer? | Storage and write budget | SARE lead | before M7 |
| <a id="q-11"></a>Q-11 | Which data protection duties apply to farm user data? | Legal compliance | SARE lead, legal | before pilot |
| <a id="q-12"></a>Q-12 | Which organisation owns hosting accounts and API keys? | Continuity after the developer leaves | SARE lead | before pilot |
| <a id="q-13"></a>Q-13 | Will automation (fans, heaters) be added later? | Would change the safety bar for alerts | SARE lead | roadmap |
| <a id="q-14"></a>Q-14 | Confirm Pinecone is dropped (tag-based KB makes it unnecessary) | Avoids an unused vendor | Prosper | now |

## Sources

Hosting and data
- Render free instances: https://render.com/docs/free
- Render Key Value: https://render.com/redis
- Turso pricing: https://turso.tech/pricing.md and billing details: https://docs.turso.tech/billing-details
- Redis Cloud Essentials plans: https://redis.io/docs/latest/operate/rc/subscriptions/view-essentials-subscription/essentials-plan-details/ and free database inactivity deletion: https://support.redislabs.com/hc/en-us/articles/33138489404818
- Upstash Redis pricing: https://upstash.com/pricing/redis.md
- Neon pricing: https://neon.com/pricing
- Pinecone pricing: https://www.pinecone.io/pricing/
- Keep-alive discussion: https://github.com/orgs/community/discussions/197645 and https://odown.com/blog/how-to-keep-a-render-free-service-from-sleeping/

Queue and ORM
- BullMQ production guide: https://docs.bullmq.io/guide/going-to-production
- BullMQ connections: https://docs.bullmq.io/guide/connections
- pg-boss: https://github.com/timgit/pg-boss
- Drizzle release notes (Turso dialect): https://newreleases.io/project/npm/drizzle-orm/release/0.34.0

AI
- Groq rate limits: https://console.groq.com/docs/rate-limits and secondary summary: https://localaimaster.com/blog/groq-api-free-guide
- Gemini rate limits: https://ai.google.dev/gemini-api/docs/rate-limits and embeddings: https://ai.google.dev/gemini-api/docs/embeddings

Hardware and domain
- MQ-135 datasheet (Hanwei): https://hobbielektronika.hu/forum/getfile.php?id=361049
- DHT11 specs: https://docs.circuit.rocks/sensors/dht11/
- ESP8266 BearSSL validation example: https://github.com/esp8266/Arduino/blob/master/libraries/ESP8266WiFi/examples/BearSSL_Validation/BearSSL_Validation.ino and HTTPS discussion: https://github.com/esp8266/Arduino/issues/3417
- ESP8266 ADC range: https://forum.arduino.cc/t/esp8266-gas-sensor-connection/376908
- Broiler ammonia, humidity and temperature: https://www.ontario.ca/page/basic-husbandry-commercial-chicken-broilers , https://pmc.ncbi.nlm.nih.gov/articles/PMC9105042 , https://openscholar.uga.edu/record/108/files/1002.pdf

Retired from the source doc (not used): Heroku student program https://heroku.com/github-students , Heroku dyno restarts https://devcenter.heroku.com/articles/dyno-restarts , Heroku Scheduler https://devcenter.heroku.com/articles/scheduler , Railway pricing https://docs.railway.com/pricing/plans.md
