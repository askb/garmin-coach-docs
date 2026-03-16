# GarminCoach v2.1 — Evidence-Based Sport Scientist

A full-stack training intelligence platform that transforms Garmin wearable data into
evidence-based coaching decisions. Every algorithm cites peer-reviewed research.

> **131 engine tests** · **20 E2E tests** · **10 integration tests** · **16/16 turbo typecheck tasks passing**

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           FRONTEND                                  │
│  Next.js 16 · Tailwind CSS · Recharts · 6-tab navigation           │
│  Pages: Today | Trends | Training | Zones | Sleep | Settings        │
│  + Onboarding, Workout Detail, AI Coach                             │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        tRPC v11 API (39+ endpoints)                 │
│  14 routers: analytics · auth · chat · garmin · journal · post      │
│              profile · readiness · sleep · workout · trends · zones  │
└──────┬──────────────┬──────────────┬──────────────┬─────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────────┐
│  Garmin    │ │  Engine    │ │  Coaching  │ │  Analytics           │
│  Service   │ │  (131 tests)│ │  Service   │ │  Service             │
│            │ │            │ │            │ │                      │
│ · OAuth    │ │ · Readiness│ │ · 16 templ │ │ · Trends/regression  │
│ · Webhooks │ │ · Strain   │ │ · Planner  │ │ · Correlations       │
│ · Backfill │ │ · ACWR     │ │ · Modulate │ │ · Notable changes    │
│ · Normalize│ │ · CTL/ATL  │ │ · Recovery │ │ · Running form       │
│            │ │ · Baselines│ │ · Sleep    │ │ · Race predictions   │
│            │ │ · Anomalies│ │   coach    │ │ · VO2max estimation  │
│            │ │ · VO2max   │ │            │ │ · Training status    │
└──────┬─────┘ └──────┬─────┘ └──────┬─────┘ └──────────┬───────────┘
       │              │              │                   │
       ▼              ▼              ▼                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                   │
│  PostgreSQL (Drizzle ORM · 13 tables) + Redis (cache/queue)         │
│  + Ollama (local AI inference for specialist agents)                │
│  Tables: Profile · DailyMetric · Activity · ReadinessScore          │
│          WeeklyPlan · DailyWorkout · ChatMessage · VO2maxEstimate   │
│          TrainingStatus · JournalEntry · CorrelationResult          │
│          RacePrediction · WorkoutTimeSeries                          │
└─────────────────────────────────────────────────────────────────────┘
```

## Features

### Engine (packages/engine — 131 tests, all evidence-based)

| Module | Description | Citation |
|--------|-------------|----------|
| Readiness scoring | Z-score based, 6 components, confidence levels, 5 zones | Buchheit 2014 |
| Strain / TRIMP | Banister impulse-response model, 0–21 scale | Banister 1991 |
| ACWR | 7d/28d acute:chronic workload ratio + EWMA variant | Hulin 2016, Williams 2017 |
| CTL/ATL/TSB | Banister fitness-fatigue model (42d/7d EMA), ramp rate | Banister 1975 |
| Load focus | Aerobic / anaerobic / mixed classification | — |
| Baselines | 14-day rolling EMA with SD, z-score infrastructure | Shaffer & Ginsberg 2017 |
| Anomalies | Z-score detection (HRV drop, RHR spike, sleep, overreaching) | — |
| VO2max estimation | ACSM running equation, Uth ratio, Cooper test | ACSM 2021, Uth 2004 |
| Race prediction | Riegel formula + VDOT-based (5K/10K/half/marathon) | Riegel 1981, Daniels 2013 |
| Training status | Productive/maintaining/detraining/overreaching/peaking/recovery | Meeusen 2013 |
| Recovery time | Strain + readiness + age + sleep debt factors | Hausswirth & Mujika 2013 |
| Sleep coach | Age/athlete/strain-adjusted need, debt tracking, bedtime recs | Mah 2011 |
| Trend analysis | Linear regression, rolling averages, notable change detection | — |
| Correlations | Pearson r with p-values for 6 metric pairs | — |
| Running form | GCT, vertical oscillation, stride length, cadence, GCT balance | Moore 2016 |
| Coaching | 16 workout templates, weekly planner, readiness-based modulation | — |

### Frontend (Next.js 16 + Tailwind + Recharts)

- **Today** — Readiness score, workout recommendation, quick stats, adjustment buttons
- **Advanced Trends** — Multi-metric overlay, trend analysis, correlations, notable changes
- **Training Load** — CTL/ATL/TSB chart, ACWR gauge, load focus, recovery, training status
- **Zone Analytics** — 7-chart dashboard: weekly zone distribution, polarization index, zone trends, efficiency trend, activity calendar, volume by week, peak performances
- **Sleep Dashboard** — Sleep stages, score, debt tracking, sleep coach, timing analysis
- **AI Coach** — 4 specialist agents (Sport Scientist, Psychologist, Nutritionist, Recovery), local Ollama inference, real-time data context
- **Settings** — Profile, Garmin connection, preferences
- **Onboarding** — 3-step setup (profile → sports/goals → schedule)
- **Workout Detail** — Structured blocks, targets, explanation

### API (tRPC v11 — 39+ endpoints across 14 routers)

`analytics (7)` · `auth (2)` · `chat (2)` · `garmin (4)` · `journal (4)` · `post (4)` · `profile (4)` · `readiness (4)` · `sleep (3)` · `workout (4)` · `trends (6)` · `zones (7)`

### Schema (Drizzle + PostgreSQL — 13 tables)

`Profile` · `DailyMetric (30+ fields)` · `Activity (25+ fields)` · `ReadinessScore` · `WeeklyPlan` · `DailyWorkout` · `ChatMessage` · `VO2maxEstimate` · `TrainingStatus` · `JournalEntry` · `CorrelationResult` · `RacePrediction` · `WorkoutTimeSeries`

## Documentation

| Document | Description |
|----------|-------------|
| [Product Specification](docs/mvp-spec.md) | Full product spec v2.0 — all features, metrics, algorithms |
| [Architecture](docs/architecture.md) | Tech stack, 14 routers, 13 tables, AI agent flow, data flow |
| [Data Model](docs/data-model.md) | Complete schema — every table, field, and relationship |
| [Engine Reference](docs/readiness-engine.md) | Every algorithm with citations, formulas, parameters |
| [Coaching Logic](docs/coaching-logic.md) | Training status, recovery, sleep coach, periodization |
| [UX Flows](docs/ux-flows.md) | All 11+ pages, charts, interactions |
| [Production Roadmap](docs/production-roadmap.md) | 16-phase roadmap — phases 1–12 complete, 13–16 pending |
| [Dev Environment](docs/dev-environment.md) | Fedora 41 / Linux setup guide |
| [Sport Science Reference](docs/sport-science-reference.md) | Peer-reviewed citations for every engine algorithm |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Monorepo** | T3 Turbo (pnpm + Turborepo) |
| **Frontend** | Next.js 16 (App Router) |
| **API** | tRPC v11 |
| **ORM** | Drizzle ORM |
| **Database** | PostgreSQL 16 |
| **Cache** | Redis |
| **Auth** | Better-Auth (Discord OAuth) |
| **Styling** | Tailwind CSS |
| **Charts** | Recharts |
| **Validation** | Zod |
| **Testing** | Vitest + Playwright |
| **CI/CD** | GitHub Actions |
| **Containers** | Docker (Postgres + Redis) |
| **AI Inference** | Ollama (local, gpt-oss:20b model) |

## Quick Start

```bash
# Prerequisites: Node.js 22+, pnpm 10+, Docker
git clone <repo-url> && cd garmin-coach
pnpm install
docker compose up -d          # Postgres + Redis
pnpm db:push                  # Apply schema (drizzle-kit push)
pnpm dev                      # Start all dev servers (turbo watch)
```

## Testing

```bash
pnpm test                     # 131 engine + 10 integration tests (Vitest)
pnpm test:e2e                 # 20 E2E tests (Playwright)
pnpm type-check               # 16/16 turbo typecheck tasks
pnpm lint                     # ESLint
```

## Project Structure

```
garmin-coach/
├── apps/
│   └── nextjs/               # Next.js 16 frontend
├── packages/
│   ├── api/                  # tRPC routers (14 routers, 39+ endpoints)
│   ├── auth/                 # Better-Auth configuration
│   ├── db/                   # Drizzle schema (13 tables)
│   ├── engine/               # Training engine (131 tests)
│   ├── ui/                   # Shared UI components
│   └── validators/           # Zod schemas
├── tooling/                  # ESLint, Prettier, TypeScript configs
├── docker-compose.yml        # Postgres + Redis
├── turbo.json
└── package.json
```

## License

Proprietary — All rights reserved.
