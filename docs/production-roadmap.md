# Production Roadmap — GarminCoach v2.1

Complete implementation roadmap for the GarminCoach Sport Scientist platform.
Phases 1–12 are **complete**. Phase 15 (AI Coach) is **complete** (v2.1).
Phases 13, 14, 16 are **pending**.

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Monorepo | T3 Turbo (pnpm + Turborepo) | Parallel builds, shared packages |
| Web | Next.js 16 (App Router) | Frontend + API routes |
| API | tRPC v11 | End-to-end typesafe (39+ endpoints) |
| ORM | Drizzle ORM | Type-safe SQL + migrations |
| Database | PostgreSQL 16 | 13 tables |
| Cache | Redis 7 | Caching, rate limiting |
| Auth | Better-Auth | Discord OAuth, sessions |
| Styling | Tailwind CSS + Recharts | UI + data visualization |
| Testing | Vitest + Playwright | Unit/integration + E2E |
| CI/CD | GitHub Actions | Automated pipeline |

---

## Monorepo Structure

```
garmin-coach/
├── apps/
│   └── nextjs/               # Next.js 16 frontend + API
├── packages/
│   ├── api/                  # tRPC routers (14 routers, 39+ endpoints)
│   ├── auth/                 # Better-Auth configuration
│   ├── db/                   # Drizzle schema (13 tables)
│   ├── engine/               # Training engine (131 tests)
│   ├── ui/                   # Shared UI components
│   └── validators/           # Zod schemas
├── tooling/                  # ESLint, Prettier, TypeScript configs
├── docker-compose.yml        # Postgres 16 + Redis 7
├── turbo.json
└── package.json
```

---

## ✅ Phase 1: Monorepo Bootstrap (Complete)

- T3 Turbo monorepo initialized
- Docker Compose for Postgres + Redis
- GitHub Actions CI pipeline (lint + typecheck + test + build)
- ESLint + Prettier + TypeScript tooling configured
- Vitest for unit/integration, Playwright for E2E

**Deliverable:** Running monorepo with CI, local DB, both apps starting.

---

## ✅ Phase 2: Database Schema & Auth (Complete)

- Drizzle schema: 13 tables defined in `packages/db/src/schema.ts`
- Better-Auth with Discord OAuth
- Auth tables: user, session, account, verification
- Application tables: Profile, DailyMetric, Activity, ReadinessScore, WeeklyPlan, DailyWorkout, ChatMessage, VO2maxEstimate, TrainingStatus, JournalEntry, CorrelationResult, RacePrediction, WorkoutTimeSeries
- Seed script for mock data (30 days)
- tRPC auth middleware (`protectedProcedure`)

**Deliverable:** Full schema deployed, seed data, auth working.

---

## ✅ Phase 3: Garmin Integration (Complete)

- OAuth 1.0a flow for Garmin Connect
- Webhook receiver for push notifications
- 30-day backfill on connection
- Normalizer: Garmin JSON → DailyMetric/Activity schema
- Idempotent sync (upsert on userId + date)

**Deliverable:** User connects Garmin → sees 30 days of data.

---

## ✅ Phase 4: Readiness & Strain Engine (Complete)

Built in `packages/engine/` with full test coverage:

- **Readiness scoring:** Z-score based, 6 components, confidence levels (Buchheit 2014)
- **TRIMP/Strain:** Banister impulse-response model, 0–21 scale (Banister 1991)
- **ACWR:** Rolling average + EWMA variant (Hulin 2016, Williams 2017)
- **CTL/ATL/TSB:** Fitness-fatigue model with ramp rate (Banister 1975)
- **Baselines:** 14-day rolling EMA with SD, population defaults (Shaffer & Ginsberg 2017)
- **Anomaly detection:** HRV drop, RHR spike, sleep disruption, overreaching

**Deliverable:** Fully tested engine, 131 tests passing.

---

## ✅ Phase 5: Coaching Logic (Complete)

- 16 workout templates (7 running, 4 cycling, 5 strength)
- Weekly planner: sport × goal × days/week selection
- Readiness-based modulation (5 zones)
- Hard day stacking prevention
- Difficulty adjustment (harder/easier)
- Volume progression with +10% cap and 4th-week deload

**Deliverable:** Full coaching engine generating personalized daily workouts.

---

## ✅ Phase 6: tRPC API Layer (Complete)

10 routers with 32+ endpoints:

| Router | Endpoints |
|--------|-----------|
| analytics | 7 — trends, correlations, running form, notable changes, training status, VO2max, race predictions |
| auth | 2 — session, sign out |
| garmin | 4 — status, OAuth, callback, backfill |
| journal | 4 — list, create, update, delete |
| post | 4 — all, byId, create, delete |
| profile | 4 — get, upsert, update, baselines |
| readiness | 4 — today, history, components, anomalies |
| sleep | 3 — dashboard, debt, coach advice |
| workout | 4 — today, adjust, week plan, detail |
| trends | 6 — summary, chart, multi-metric, comparison, CTL/ATL/TSB, ACWR |

**Deliverable:** Complete API with integration tests.

---

## ✅ Phase 7: Frontend — Web App (Complete)

7+ pages built with Next.js 16 + Tailwind + Recharts:

- **Today/Home:** Readiness circle, workout card, quick stats, adjustment buttons
- **Advanced Trends:** Multi-metric overlay, trend regression, correlations, notable changes
- **Training Load:** CTL/ATL/TSB chart, ACWR gauge, load focus, training status, recovery
- **Zone Analytics:** 7-chart dashboard — zone distribution, polarization, efficiency, calendar (NEW v2.1)
- **Sleep Dashboard:** Sleep stages, score, debt, coach, timing analysis
- **AI Coach:** 4 specialist agents via local Ollama inference (NEW v2.1)
- **Onboarding:** 3-step flow (profile → sports → schedule)
- **Settings:** Profile, Garmin, preferences
- **Workout Detail:** Structured blocks, targets, explanation

6-tab bottom navigation: Today · Trends · Training · Zones · Sleep · Settings

**Deliverable:** Fully functional web app with E2E tests (20 Playwright tests).

---

## ✅ Phase 8: Extended Engine Modules (Complete)

Additional engine modules beyond core readiness/strain:

- **VO2max estimation:** ACSM, Uth, Cooper — 3 methods (ACSM 2021, Uth 2004, Cooper 1968)
- **Race prediction:** Riegel + VDOT for 5K/10K/half/marathon (Riegel 1981, Daniels 2013)
- **Training status:** 6-state classification (Meeusen 2013)
- **Recovery time:** Modifiers for strain, readiness, age, sleep debt (Hausswirth & Mujika 2013)
- **Sleep coach:** Sleep need, debt tracking, bedtime recommendations (Mah 2011)
- **Trend analysis:** Linear regression, rolling averages, notable changes
- **Correlations:** Pearson r with p-values for 6 metric pairs
- **Running form:** GCT, VO, stride length, cadence, GCT balance (Moore 2016)
- **Load focus:** Aerobic/anaerobic/mixed classification

**Deliverable:** All engine modules with evidence-based citations, 131 tests.

---

## ✅ Phase 9: Testing & Quality (Complete)

| Category | Count | Tool |
|----------|-------|------|
| Engine unit tests | 131 | Vitest |
| E2E tests | 20 | Playwright |
| Integration tests | 10 | Vitest |
| Type checking | 16/16 | TypeScript (turbo) |
| **Total** | **161** | |

**Deliverable:** All tests green, 16/16 turbo typecheck tasks passing.

---

## ✅ Phase 10: CI/CD Pipeline (Complete)

- GitHub Actions: lint + typecheck + test + build on push/PR
- Docker Compose for local development
- Drizzle-kit for schema migrations

**Deliverable:** Automated CI pipeline.

---

## ✅ Phase 11: Sport Science Documentation (Complete)

- Comprehensive sport science reference document
- Every engine algorithm cited with peer-reviewed research
- Validation methodology documented
- Limitations and disclaimers clearly stated

**Deliverable:** [Sport Science Reference](sport-science-reference.md)

---

## ✅ Phase 12: Documentation Overhaul (Complete)

- Full product spec v2.0
- Architecture documentation with all 10 routers, 13 tables
- Complete data model with every field documented
- Engine reference with all algorithms and citations
- Updated coaching logic with training status, recovery, sleep coach
- UX flows for all 7+ pages

**Deliverable:** Comprehensive documentation suite.

---

## ✅ Phase 12a: Zone Analytics Dashboard (Complete — v2.1)

7-chart zone analytics dashboard with sport filtering and polarization tracking.

### Implemented Features

- **Weekly zone distribution:** Stacked bar chart — HR zone minutes by ISO week
- **Polarization index:** Seiler's formula: PI = ln(1/Σpi²), composed chart
- **Zone trends:** 100% stacked area chart — monthly zone % breakdown
- **Efficiency trend:** Scatter chart — pace/HR efficiency index per activity
- **Activity calendar:** CSS grid heatmap — daily activity intensity
- **Volume by week:** Stacked bar — weekly minutes by sport type
- **Peak performances:** Monthly bests for pace, distance, duration, HR
- **Sport filtering:** All charts filterable by sport type

### Technical Implementation

- New page: `/zones`
- New tRPC router: `zones` (7 endpoints)
- `hr_zone_minutes` JSONB field on Activity table
- Recharts for all chart visualizations

**Deliverable:** Full zone analytics dashboard with 7 chart sections.

---

## ✅ Phase 12b: Docker Tuning & Health Monitoring (Complete — v2.1)

### Docker Compose Tuning

- PostgreSQL: memory limit 2GB, CPU 2.0 cores, shared_buffers=256MB, effective_cache_size=1GB, work_mem=16MB, max_connections=50, slow query logging (>1s)
- Redis: memory limit 256MB, CPU 0.5 cores, maxmemory 128MB with allkeys-lru eviction
- Both containers: `restart: unless-stopped`

### Health Monitoring

- `scripts/health-check.sh` — comprehensive system health checker
- Checks: memory, disk, Docker containers, PostgreSQL data, Redis memory, Next.js, power/battery, NVMe
- Crash history classification from journalctl boot records
- Recommendations mode (`--full`) for containers without limits, s2idle warnings

### Data Quality Fixes

- Removed 4 seed VO2max contamination values (47–50.5 → real max is 33.2)
- Removed duplicate VO2max entries
- Updated profile to real user stats: age=48, mass_kg=115, resting_hr=68, vo2max_running=32.9, max_hr=172
- Hydration mismatch fix: replaced `toLocaleDateString()` with deterministic formatters

**Deliverable:** Production-ready resource governance and monitoring.

---

## 🔲 Phase 13: Activity Detail Page (Pending)

Individual activity view with rich data visualization.

### Planned Features

- **Time-series charts:** HR, pace, cadence, altitude over activity duration
- **Split analysis:** Per-km or per-mile splits with pace/HR
- **HR zone distribution:** Time-in-zone bar chart
- **Running form metrics:** GCT, vertical oscillation, stride length (if available)
- **Activity summary:** Distance, duration, avg/max HR, calories, TRIMP, strain
- **Map view:** GPS track on map (if coordinates available)
- **Comparison:** Overlay with previous similar activities

### Technical Requirements

- New page: `/workout/[id]` (enhanced from current basic detail)
- Uses `WorkoutTimeSeries` table for time-series data
- Recharts for all visualizations
- tRPC endpoint: `analytics.getActivityDetail({ activityId })`

---

## 🔲 Phase 14: Journal (Pending)

Daily athlete journal with mood tracking and metric correlation.

### Planned Features

- **Daily entries:** Free-form text with date association
- **Mood tracking:** 5-point scale (great/good/neutral/tired/poor)
- **Tags:** Customizable tags (race, travel, injury, stress, etc.)
- **Correlation view:** How mood/tags correlate with readiness, HRV, sleep
- **Timeline:** Scrollable journal feed with metric annotations

### Technical Requirements

- New page: `/journal`
- Uses `JournalEntry` table (already in schema)
- tRPC router: `journal` (4 endpoints already defined)
- Add 6th navigation option or integrate into Settings

---

## ✅ Phase 15: AI Coach — Specialist Agents (Complete — v2.1)

AI-powered coaching chat with 4 specialist agents using local Ollama inference.

### Implemented Features

- **4 specialist agents:** Sport Scientist, Sport Psychologist, Nutritionist, Recovery Specialist
- **Local AI inference:** Ollama with gpt-oss:20b model — zero cost, fully private
- **Real-time data context:** Last 14 days of metrics, activities, zones, readiness gathered from PostgreSQL
- **Architecture:** Browser → tRPC → data context builder → system prompt → Ollama HTTP API → response
- **Fallback:** Data-driven template responses when Ollama is unavailable
- **Agent selector tabs:** Switch between specialist personas
- **Quick prompt chips:** Pre-built questions for common queries
- **Markdown rendering:** Responses rendered with headers, lists, emphasis
- **Typing indicator:** Loading state while Ollama generates

### Technical Implementation

- `packages/api/src/lib/ollama.ts` — Ollama HTTP client
- `packages/api/src/lib/agent-prompts.ts` — 4 specialist system prompts
- `packages/api/src/lib/data-context.ts` — Real-time data context builder
- `packages/api/src/router/chat.ts` — tRPC chat router (2 endpoints)
- Uses `ChatMessage` table for conversation history

**Deliverable:** Fully functional AI coaching with local inference.

---

## 🔲 Phase 16: VO2max & Predictions Page (Pending)

Dedicated page for performance estimation and race planning.

### Planned Features

- **VO2max history chart:** Track VO2max estimates over time with method indicators
- **Race prediction calculator:** Input a recent race → get predictions for other distances
- **Training paces:** VDOT-derived pace zones for easy/tempo/threshold/interval
- **Fitness trend:** VO2max trend line with improvement rate
- **Comparison:** Side-by-side Riegel vs VDOT predictions

### Technical Requirements

- New page: `/predictions`
- Uses `VO2maxEstimate` and `RacePrediction` tables (already in schema)
- tRPC endpoints: `analytics.getVO2maxHistory`, `analytics.getRacePredictions` (already defined)
- Recharts for VO2max trend chart

---

## Dependency Graph

```
Phase 1 (Bootstrap)
 └── Phase 2 (Schema & Auth)
      └── Phase 3 (Garmin Integration)
           └── Phase 4 (Readiness Engine)
                └── Phase 5 (Coaching Logic)
                     └── Phase 6 (tRPC API)
                          ├── Phase 7 (Web App)
                          └── Phase 8 (Extended Engine)
                               └── Phase 9 (Testing)
                                    └── Phase 10 (CI/CD)
                                         └── Phase 11 (Science Docs)
                                              └── Phase 12 (Docs Overhaul)
                                                   ├── Phase 12a (Zone Analytics) ← v2.1
                                                   ├── Phase 12b (Docker/Health)  ← v2.1
                                                   └── Phase 15 (AI Agents)       ← v2.1

Remaining (can be done in parallel):
├── Phase 13 (Activity Detail)
├── Phase 14 (Journal)
└── Phase 16 (VO2max & Predictions Page)
```

---

## Effort Summary

| Phase | Status | Focus |
|-------|--------|-------|
| 1: Bootstrap | ✅ Complete | Monorepo, CI, Docker |
| 2: Schema & Auth | ✅ Complete | 13 tables, Better-Auth |
| 3: Garmin | ✅ Complete | OAuth, webhooks, backfill |
| 4: Readiness Engine | ✅ Complete | Score, strain, ACWR, baselines, anomalies |
| 5: Coaching Logic | ✅ Complete | 16 templates, planner, modulation |
| 6: tRPC API | ✅ Complete | 14 routers, 39+ endpoints |
| 7: Web App | ✅ Complete | 11+ pages, Recharts, E2E tests |
| 8: Extended Engine | ✅ Complete | VO2max, race pred, training status, sleep, trends |
| 9: Testing | ✅ Complete | 161 total tests |
| 10: CI/CD | ✅ Complete | GitHub Actions pipeline |
| 11: Science Docs | ✅ Complete | Sport science reference |
| 12: Docs Overhaul | ✅ Complete | Full documentation suite |
| 12a: Zone Analytics | ✅ Complete | 7-chart zone dashboard, polarization (v2.1) |
| 12b: Docker/Health | ✅ Complete | Resource limits, health-check.sh (v2.1) |
| 13: Activity Detail | 🔲 Pending | Time-series charts, splits, maps |
| 14: Journal | 🔲 Pending | Mood tracking, tags, correlations |
| 15: AI Coach Agents | ✅ Complete | 4 specialist agents, local Ollama (v2.1) |
| 16: VO2max & Predictions | 🔲 Pending | VO2max chart, race calculator, paces |

---

## Production Recommendations (v2.1)

### AI Infrastructure

For production Ollama deployment:

| Concern | Recommendation |
|---------|---------------|
| **Model serving** | Dedicated GPU instance (NVIDIA T4/A10G) or quantized CPU model |
| **Scaling** | Ollama behind a load balancer with multiple model instances |
| **Fallback** | Template response system already implemented — graceful degradation |
| **Latency** | Expect 2–8s for 20B model; consider smaller quantized variants for <1s responses |
| **Privacy** | Keep inference local/on-prem — zero data leaves the infrastructure |
| **Model updates** | Pin model versions; test before rolling updates |

### Monitoring & Alerting

Based on crash history analysis from `scripts/health-check.sh`:

| Signal | Alert Threshold | Action |
|--------|----------------|--------|
| Container OOM kills | Any occurrence | Increase memory limit or investigate leak |
| PostgreSQL slow queries | > 1s (already logged) | Add indexes or optimize query |
| Redis memory > 90% maxmemory | Approaching 128MB | Review eviction patterns |
| Ollama response time | > 10s p95 | Consider model size reduction |
| Boot crash frequency | > 2 unclean shutdowns/week | Investigate power/kernel issues |
| Disk usage | > 85% on any volume | Clean Docker images, rotate logs |

### Docker Resource Governance

| Service | Current Limit | Production Recommendation |
|---------|--------------|--------------------------|
| PostgreSQL | 2GB RAM / 2 CPU | 4–8GB RAM for production workloads |
| Redis | 256MB RAM / 0.5 CPU | Sufficient for single-user; scale with users |
| Ollama | Unbounded | 8GB+ RAM for 20B model; GPU preferred |
| Next.js | Unbounded | 1–2GB RAM limit recommended |
