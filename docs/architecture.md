# Architecture & Tech Stack — GarminCoach v2.1

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CLIENTS                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │ Next.js 16 Web App (Tailwind + Recharts)                     │  │
│  │ 6 tabs: Today · Trends · Training · Zones · Sleep · Settings │  │
│  │ + Onboarding · Workout Detail · AI Coach                     │  │
│  └──────────────────────────┬────────────────────────────────────┘  │
│                             │                                       │
│  ┌──────────────────────────┴────────────────────────────────────┐  │
│  │ Garmin Webhooks (Push from Garmin Connect)                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    tRPC v11 API LAYER (39+ endpoints)                │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    Next.js API Routes                          │ │
│  │                    (tRPC Router — appRouter)                   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  14 Routers:                                                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │analytics │ │  auth    │ │   chat   │ │  garmin  │ │ journal  │ │
│  │  (7)     │ │  (2)    │ │  (2)    │ │  (4)    │ │  (4)    │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │   post   │ │ profile  │ │readiness │ │  sleep   │ │  trends  │ │
│  │  (4)     │ │  (4)    │ │  (4)    │ │  (3)    │ │   (6)   │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘ │
│  ┌──────────┐ ┌──────────┐                                         │
│  │ workout  │ │  zones   │                                         │
│  │  (4)     │ │  (7)    │                                         │
│  └──────────┘ └──────────┘                                         │
└──────┬──────────────┬──────────────┬──────────────┬─────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌────────────┐ ┌─────────────┐ ┌────────────┐ ┌──────────────────────┐
│  Garmin    │ │   Engine    │ │  Coaching  │ │  Analytics           │
│  Service   │ │ (131 tests) │ │  Service   │ │  Service             │
│            │ │             │ │            │ │                      │
│ · OAuth 1.0a│ · Readiness │ │ · 16 templ.│ │ · Trend regression   │
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
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ PostgreSQL 16    │  │ Redis 7          │  │ Ollama           │  │
│  │ (Drizzle ORM)    │  │ (Cache / Queue)  │  │ (Local AI / LLM) │  │
│  │ 13 tables        │  │                  │  │ gpt-oss:20b      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### 2.1 Core Stack

| Layer | Technology | Version | Purpose |
|-------|-----------|---------|---------|
| **Monorepo** | Turborepo + pnpm | — | Parallel builds, shared packages |
| **Framework** | Next.js | 16 | App Router, SSR, API routes |
| **API** | tRPC | v11 | End-to-end type safety |
| **ORM** | Drizzle ORM | — | Type-safe SQL, migrations (drizzle-kit) |
| **Auth** | Better-Auth | — | Discord OAuth, session management |
| **Styling** | Tailwind CSS | — | Utility-first CSS |
| **Charts** | Recharts | — | Training load, trends, sleep visualizations |
| **Validation** | Zod | — | Runtime + compile-time validation |

### 2.2 Infrastructure

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Database** | PostgreSQL 16 | Primary data store (13 tables) |
| **Cache** | Redis 7 | Caching, rate limiting |
| **AI Inference** | Ollama (local) | LLM for specialist coaching agents (gpt-oss:20b) |
| **Containers** | Docker Compose | Local dev (Postgres + Redis) |
| **CI/CD** | GitHub Actions | Lint + typecheck + test + build |
| **Runtime** | Node.js 22+ | Server runtime |
| **Package manager** | pnpm 10+ | Fast, disk-efficient |

---

## 3. Package Architecture

### 3.1 Monorepo Structure

```
garmin-coach/
├── apps/
│   └── nextjs/                  # Next.js 16 frontend + API routes
│       ├── src/app/             # App Router pages
│       │   ├── (app)/           # Authenticated pages
│       │   │   ├── page.tsx     # Today / Home
│       │   │   ├── trends/      # Advanced Trends
│       │   │   ├── training/    # Training Load
│       │   │   ├── zones/       # Zone Analytics Dashboard (NEW)
│       │   │   ├── sleep/       # Sleep Dashboard
│       │   │   ├── settings/    # Settings
│       │   │   └── workout/     # Workout Detail
│       │   └── onboarding/      # 3-step setup flow
│       └── e2e/                 # Playwright E2E tests (20 tests)
├── packages/
│   ├── api/                     # tRPC routers (14 routers, 39+ endpoints)
│   │   └── src/
│   │       ├── root.ts          # appRouter definition
│   │       ├── lib/
│   │       │   ├── ollama.ts    # Ollama HTTP client (AI inference)
│   │       │   ├── agent-prompts.ts # 4 specialist agent system prompts
│   │       │   └── data-context.ts  # Real-time data context builder
│   │       └── router/
│   │           ├── analytics.ts # 7 endpoints
│   │           ├── auth.ts      # 2 endpoints
│   │           ├── chat.ts      # 2 endpoints (AI specialist agents)
│   │           ├── garmin.ts    # 4 endpoints
│   │           ├── journal.ts   # 4 endpoints
│   │           ├── post.ts      # 4 endpoints
│   │           ├── profile.ts   # 4 endpoints
│   │           ├── readiness.ts # 4 endpoints
│   │           ├── sleep.ts     # 3 endpoints
│   │           ├── workout.ts   # 4 endpoints
│   │           ├── trends.ts    # 6 endpoints
│   │           └── zones.ts     # 7 endpoints (zone analytics)
│   ├── auth/                    # Better-Auth configuration
│   ├── db/                      # Drizzle schema + client (13 tables)
│   │   └── src/
│   │       ├── schema.ts        # All table definitions
│   │       ├── auth-schema.ts   # Better-Auth managed tables
│   │       └── client.ts        # Database client
│   ├── engine/                  # Training engine (131 tests)
│   │   └── src/
│   │       ├── readiness/       # Z-score readiness scoring
│   │       ├── strain/          # TRIMP, strain, ACWR, CTL/ATL/TSB
│   │       ├── baselines/       # Rolling EMA, population defaults
│   │       ├── anomalies/       # HRV drop, RHR spike, sleep, overreaching
│   │       ├── vo2max/          # ACSM, Uth, Cooper estimation
│   │       ├── racePrediction/  # Riegel + VDOT predictions
│   │       ├── trainingStatus/  # 6-state classification
│   │       ├── recovery/        # Recovery time estimation
│   │       ├── sleep/           # Sleep coach, debt tracking
│   │       ├── trends/          # Regression, rolling averages
│   │       ├── correlations/    # Pearson r with p-values
│   │       ├── runningForm/     # GCT, VO, stride, cadence, balance
│   │       ├── coaching/        # Templates, planner, modulation
│   │       │   └── templates/   # 16 workout templates
│   │       ├── types.ts         # Shared interfaces
│   │       └── index.ts         # Public API exports
│   ├── ui/                      # Shared UI components
│   └── validators/              # Zod schemas (shared validation)
├── tooling/
│   ├── eslint/                  # Shared ESLint config
│   ├── prettier/                # Shared Prettier config
│   └── typescript/              # Shared tsconfig
├── docker-compose.yml           # Postgres 16 + Redis 7
├── turbo.json
└── package.json
```

### 3.2 Engine Module Architecture

```
packages/engine/src/
├── readiness/          # calculateReadiness() → score, zone, components, confidence
├── strain/             # calculateTRIMP(), calculateStrain(), calculateACWR(),
│                       # calculateCTL(), calculateATL(), calculateTSB()
├── baselines/          # calculateBaseline(), getPopulationDefaults(), blendBaselines()
├── anomalies/          # detectAnomalies() → HRV drop, RHR spike, sleep, overreaching
├── vo2max/             # estimateVO2max() via ACSM, Uth, Cooper
├── racePrediction/     # predictRaceTime() via Riegel + VDOT
├── trainingStatus/     # classifyTrainingStatus() → 6 states
├── recovery/           # estimateRecoveryTime() with modifiers
├── sleep/              # calculateSleepNeed(), trackSleepDebt(), recommendBedtime()
├── trends/             # analyzeTrend() → regression, rolling avg, notable changes
├── correlations/       # calculateCorrelation() → Pearson r + p-value
├── runningForm/        # analyzeRunningForm() → GCT, VO, stride, cadence, balance
├── coaching/           # selectWeeklyTemplate(), modulateWorkout(), generateDailyWorkout()
│   └── templates/      # 16 workout templates (running, cycling, strength)
├── types.ts            # Shared TypeScript interfaces
└── index.ts            # Public API surface
```

---

## 4. API Design (tRPC Routers)

### 4.1 Router Structure

```typescript
// packages/api/src/root.ts
export const appRouter = createTRPCRouter({
  analytics: analyticsRouter,   // 7 endpoints
  auth:      authRouter,        // 2 endpoints
  chat:      chatRouter,        // 2 endpoints (AI specialist agents)
  garmin:    garminRouter,      // 4 endpoints
  journal:   journalRouter,     // 4 endpoints
  post:      postRouter,        // 4 endpoints
  profile:   profileRouter,     // 4 endpoints
  readiness: readinessRouter,   // 4 endpoints
  sleep:     sleepRouter,       // 3 endpoints
  workout:   workoutRouter,     // 4 endpoints
  trends:    trendsRouter,      // 6 endpoints
  zones:     zonesRouter,       // 7 endpoints (zone analytics)
});
```

### 4.2 Endpoint Catalog

```typescript
// analytics router (7)
analytics.getTrendAnalysis({ metric, days })        // → TrendResult (regression + direction)
analytics.getCorrelations({ period })               // → CorrelationResult[] (6 pairs)
analytics.getNotableChanges({ days })               // → NotableChange[]
analytics.getRunningForm({ activityId })            // → RunningFormAnalysis
analytics.getTrainingStatus()                       // → TrainingStatus classification
analytics.getVO2maxHistory({ days })                // → VO2maxEstimate[]
analytics.getRacePredictions({ inputRace })          // → RacePrediction[] (5K/10K/half/marathon)

// auth router (2)
auth.getSession()                                    // → Session | null
auth.signOut()                                       // → void

// garmin router (4)
garmin.getConnectionStatus()                         // → { connected, lastSync }
garmin.initiateOAuth()                               // → { authUrl }
garmin.handleCallback({ code })                      // → store tokens
garmin.triggerBackfill({ days })                     // → { metricsInserted, activitiesInserted }

// journal router (4)
journal.list({ days })                               // → JournalEntry[]
journal.create({ content, mood, tags })              // → JournalEntry
journal.update({ id, content, mood, tags })          // → JournalEntry
journal.delete({ id })                               // → void

// post router (4)
post.all()                                           // → Post[]
post.byId({ id })                                    // → Post
post.create({ title, content })                      // → Post
post.delete({ id })                                  // → void

// profile router (4)
profile.get()                                        // → Profile
profile.upsert({ ... })                              // → Profile
profile.update({ ... })                              // → Profile
profile.getBaselines()                               // → Baselines

// readiness router (4)
readiness.getToday()                                 // → { score, zone, explanation, components, confidence }
readiness.getHistory({ days })                       // → ReadinessScore[]
readiness.getComponents({ date })                    // → { sleep, hrv, rhr, load, stress }
readiness.getAnomalies()                             // → AnomalyAlert[]

// sleep router (3)
sleep.getDashboard({ days })                         // → { scores, stages, debt, efficiency }
sleep.getDebt()                                      // → { current, trend, recommendation }
sleep.getCoachAdvice()                               // → { bedtime, sleepNeed, tips }

// workout router (4)
workout.getToday()                                   // → DailyWorkout
workout.adjustDifficulty({ direction })              // → WorkoutRecommendation
workout.getWeekPlan()                                // → DailyWorkout[]
workout.getDetail({ id })                            // → { structure, targets, explanation }

// trends router (6)
trends.getSummary({ period })                        // → { readiness, strain, sleep, hrv, rhr, steps }
trends.getChart({ metric, days })                    // → DataPoint[]
trends.getMultiMetric({ metrics, days })             // → { [metric]: DataPoint[] }
trends.getComparison({ period1, period2 })           // → { deltas, significance }
trends.getCTLATLTSB({ days })                        // → { ctl[], atl[], tsb[] }
trends.getACWR({ days })                             // → { values[], zone }

// zones router (7) — NEW: Zone Analytics Dashboard
zones.getWeeklyZoneDistribution({ weeks, sport? })   // → HR zone minutes grouped by ISO week
zones.getPolarizationIndex({ weeks, sport? })          // → Seiler's polarization index (easy/mod/hard)
zones.getZoneTrends({ months })                        // → Monthly zone % breakdown
zones.getEfficiencyTrend({ days, sport? })             // → Pace/HR efficiency index per activity
zones.getActivityCalendar({ year })                    // → Daily activity heatmap data
zones.getVolumeByWeek({ weeks })                       // → Weekly minutes by sport type
zones.getPeakPerformances({ months })                  // → Monthly bests for pace/distance/duration/HR

// chat router (2) — NEW: AI Specialist Agents
chat.sendMessage({ content, agent })                   // → AI response via Ollama (or fallback)
chat.getHistory({ limit? })                            // → ChatMessage[]
```

---

## 5. Data Flow

```
1. User syncs Garmin watch → Garmin Connect cloud
2. Garmin Connect → Webhook POST → /api/garmin/webhook
3. Webhook handler:
   a. Verify signature
   b. Parse payload type (dailies, activities, sleep, stress)
   c. Normalize Garmin JSON → DailyMetric / Activity schema
   d. Upsert into PostgreSQL (idempotent on userId + date)
4. Engine pipeline triggers:
   a. Recalculate baselines (14-day rolling EMA)
   b. Compute readiness score (6 components → weighted composite)
   c. Detect anomalies (HRV, RHR, sleep, overreaching)
   d. Update training status classification
   e. Calculate CTL/ATL/TSB and ACWR
   f. Estimate recovery time
5. Coaching pipeline:
   a. Generate/modulate today's workout based on readiness zone
   b. Update weekly plan if needed
6. Frontend fetches via tRPC:
   a. Today: readiness + workout + quick stats
   b. Trends: multi-metric charts + correlations
   c. Training: CTL/ATL/TSB + ACWR + load focus
   d. Sleep: stages + debt + coach advice
7. User trains → Garmin records → cycle repeats
```

---

## 5a. AI Specialist Agent Flow

```
Browser → tRPC chat.sendMessage({ content, agent })
  │
  ▼
Data Context Builder (packages/api/src/lib/data-context.ts)
  │  Gathers from PostgreSQL:
  │  · Last 14 days of DailyMetric (sleep, HRV, RHR, stress, Body Battery)
  │  · Recent activities (sport, duration, strain, zones)
  │  · Current readiness score and zone
  │  · Profile (age, mass, goals, experience)
  │
  ▼
System Prompt Selection (packages/api/src/lib/agent-prompts.ts)
  │  4 specialist personas:
  │  · Sport Scientist — periodization, ACWR, VO2max, zone recommendations
  │  · Sport Psychologist — motivation, mental resilience, consistency, race-day prep
  │  · Nutritionist — fueling strategies, recovery nutrition, calorie/macro guidance
  │  · Recovery Specialist — sleep optimization, deload protocols, injury risk, HRV
  │
  ▼
Ollama HTTP API (packages/api/src/lib/ollama.ts)
  │  POST http://localhost:11434/api/generate
  │  Model: gpt-oss:20b
  │  Fully local, zero cost, private
  │
  ▼
Response → tRPC → Browser (markdown rendering, typing indicator)

Fallback: If Ollama is unavailable, returns data-driven template responses
built from the same real-time context data.
```

---

## 5b. Docker Resource Limits

| Service | Memory Limit | CPU Limit | Key Tuning |
|---------|-------------|-----------|------------|
| PostgreSQL | 2 GB | 2.0 cores | shared_buffers=256MB, effective_cache_size=1GB, work_mem=16MB, max_connections=50, slow query logging (>1s) |
| Redis | 256 MB | 0.5 cores | maxmemory 128MB, allkeys-lru eviction |

Both containers run with `restart: unless-stopped`.

---

## 6. Security & Privacy

### 6.1 Authentication

- Better-Auth with Discord OAuth provider
- Database-backed sessions with httpOnly cookies
- tRPC middleware validates session on every protected procedure
- Garmin OAuth 1.0a tokens encrypted at rest

### 6.2 Data Protection

- HTTPS everywhere (TLS 1.3)
- Database connections via SSL
- Row-level security: users access only their own data
- Input validation on all mutations (Zod schemas)
- Rate limiting on public endpoints

### 6.3 Privacy

- Explicit consent before Garmin connection
- Transparent scoring: users see exactly how readiness is computed
- Data export capability (GDPR: right to portability)
- Account deletion removes all data (GDPR: right to erasure)
- No data sold or shared with third parties

---

## 7. Background Jobs

| Job | Trigger | Frequency |
|-----|---------|-----------|
| Garmin data sync | Webhook push | Real-time |
| Readiness computation | After sync | On data arrival |
| Baseline recalculation | After new data | Daily |
| Anomaly detection | After readiness computed | Daily |
| Weekly plan generation | Sunday night or on demand | Weekly |
| Daily workout modulation | After readiness computed | Daily |
| CTL/ATL/TSB update | After activity sync | On data arrival |

### Event Pipeline

```
Garmin webhook received
  → Normalize & upsert data
  → Recalculate baselines
  → Compute readiness score
  → Detect anomalies
  → Update training status
  → Modulate today's workout
```

---

## 8. Deployment

### 8.1 Development

```bash
docker compose up -d          # PostgreSQL 16 + Redis 7
pnpm install
pnpm db:push                  # Apply Drizzle schema
pnpm dev                      # Turbo watch — all packages
```

### 8.2 CI/CD (GitHub Actions)

```yaml
on: [push, pull_request]
jobs:
  check:
    steps:
      - pnpm install --frozen-lockfile
      - pnpm lint
      - pnpm type-check        # 16/16 tasks
      - pnpm test               # 131 engine + 10 integration
      - pnpm test:e2e           # 20 Playwright tests
      - pnpm build
```

### 8.3 Environments

| Environment | URL | Purpose |
|-------------|-----|---------|
| Development | localhost:3000 | Local dev |
| Preview | pr-{n}.vercel.app | PR previews |
| Production | garmincoach.app | Live |
