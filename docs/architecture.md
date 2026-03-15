# Architecture & Tech Stack — GarminCoach

## 1. High‑Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTS                               │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │ Mobile App   │  │ Web App      │  │ Garmin Webhooks   │  │
│  │ (Expo/RN)    │  │ (Next.js)    │  │ (Push from Garmin)│  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘  │
│         │                 │                    │             │
└─────────┼─────────────────┼────────────────────┼─────────────┘
          │                 │                    │
          ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                     API GATEWAY / tRPC                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                   Next.js API Routes                  │   │
│  │                   (tRPC Router)                       │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────┬──────────────┬──────────────┬────────────────────┘
           │              │              │
           ▼              ▼              ▼
┌──────────────┐ ┌───────────────┐ ┌──────────────────┐
│ Garmin       │ │ Readiness &   │ │ Coaching         │
│ Ingestion    │ │ Strain Engine │ │ Service           │
│ Service      │ │               │ │                  │
│ - OAuth      │ │ - Score calc  │ │ - Weekly planner │
│ - Webhook RX │ │ - Baselines   │ │ - Template lib   │
│ - Backfill   │ │ - Anomalies   │ │ - Chat handler   │
│ - Normalize  │ │ - Zones       │ │ - Modulation     │
└──────┬───────┘ └───────┬───────┘ └────────┬─────────┘
       │                 │                   │
       ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ PostgreSQL   │  │ Redis        │  │ Object Storage   │  │
│  │ (Primary DB) │  │ (Cache/Queue)│  │ (Garmin raw JSON)│  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Tech Stack

### 2.1 T3 Stack Foundation

| Layer | Technology | Why |
|-------|-----------|-----|
| **Framework** | [create-t3-app](https://create.t3.gg/) | Full‑stack TypeScript, batteries included |
| **Runtime** | Next.js 14+ (App Router) | SSR, API routes, edge functions |
| **API** | tRPC v11 | End‑to‑end type safety, no codegen |
| **ORM** | Drizzle ORM | Type‑safe, SQL‑like, great migrations |
| **Auth** | Better-Auth | OAuth providers, session management, Expo support |
| **Styling** | Tailwind CSS + shadcn/ui | Rapid UI development |
| **Validation** | Zod | Runtime + compile‑time validation |

### 2.2 Infrastructure

| Component | Technology | Why |
|-----------|-----------|-----|
| **Database** | PostgreSQL 16 (Neon / Supabase) | Relational + JSONB for flexibility |
| **Cache/Queue** | Redis (Upstash) | Serverless‑friendly, rate limiting |
| **Mobile** | Expo + React Native | Share logic with web, native performance |
| **Hosting** | Vercel (web) / Fly.io (workers) | Edge‑first, auto‑scaling |
| **Monitoring** | Sentry + PostHog | Error tracking + product analytics |
| **CI/CD** | GitHub Actions | Automated testing, preview deploys |

### 2.3 AI Layer

| Component | Technology | Why |
|-----------|-----------|-----|
| **LLM** | OpenAI GPT‑4o‑mini | Cost‑effective for structured coaching responses |
| **SDK** | Vercel AI SDK | Streaming, structured output, tool calling |
| **Prompt mgmt** | In‑code templates | Simple, version‑controlled |

---

## 3. Service Architecture

### 3.1 Garmin Ingestion Service

```
garmin-ingestion/
├── oauth/           # OAuth 1.0a flow for Garmin Connect
├── webhooks/        # Receive push notifications from Garmin
├── backfill/        # Pull historical data (30 days)
├── normalize/       # Map Garmin JSON → our data model
└── sync/            # Incremental daily sync job
```

**Garmin API Integration:**
- OAuth 1.0a consumer (Garmin still uses OAuth 1.0a)
- Webhook receiver for real‑time push data
- Pull‑based backfill for historical data
- Rate limiting: respect Garmin API limits (60 req/min)

### 3.2 Readiness Engine

```
packages/engine/src/
├── readiness/
│   └── index.ts          # All component scorers + calculateReadiness()
├── strain/
│   └── index.ts          # TRIMP, strain score, ACWR, consecutive hard days
├── baselines/
│   └── index.ts          # EMA, population defaults, personal baselines
├── anomalies/
│   └── index.ts          # HRV crash, RHR spike, sleep deficit, overreaching
├── coaching/
│   ├── index.ts          # Weekly planner, modulation, daily workout generation
│   └── templates/
│       └── index.ts      # 20+ workout templates (running, cycling, strength)
├── types.ts              # Shared interfaces
└── index.ts              # Public API exports
```

### 3.3 Coaching Service

```
packages/engine/src/coaching/
├── index.ts              # selectWeeklyTemplate(), modulateWorkout(),
│                         # generateDailyWorkout(), adjustDifficulty()
└── templates/
    └── index.ts          # WorkoutTemplate interface + all templates
                          # (runningTemplates, cyclingTemplates, strengthTemplates)
```

---

## 4. API Design (tRPC Routers)

### 4.1 Router Structure

```typescript
// src/server/api/root.ts
export const appRouter = createTRPCRouter({
  auth: authRouter,
  garmin: garminRouter,
  post: postRouter,
  profile: profileRouter,
  readiness: readinessRouter,
  workout: workoutRouter,
  trends: trendsRouter,
});
```

### 4.2 Key Procedures

```typescript
// garmin router
garmin.getConnectionStatus()       // → { connected, lastSync }
garmin.initiateOAuth()             // → { authUrl }
garmin.handleCallback()            // ← code → store tokens
garmin.triggerBackfill({ days })   // → { metricsInserted, activitiesInserted }

// readiness router
readiness.getToday()               // → { score, zone, explanation, factors }
readiness.getHistory({ days })     // → ReadinessScore[]
readiness.getComponents({ date })  // → { sleep, hrv, rhr, load, stress }
readiness.getAnomalies()           // → AnomalyAlert[]

// workout router
workout.getToday()                 // → { workout }
workout.adjustDifficulty({ direction: 'harder' | 'easier' })  // → { adjustedWorkout }
workout.getWeekPlan()              // → DailyWorkout[]
workout.getDetail({ id })         // → { structure, targets, explanation }

// chat router — Planned (not yet implemented)
// chat.sendMessage({ content })   // → { response, context }
// chat.getHistory({ limit })      // → ChatMessage[]

// post router (legacy)
post.all()                         // → Post[]
post.byId({ id })                 // → Post
post.create({ title, content })   // → Post
post.delete({ id })               // → void

// trends router
trends.getSummary({ period })      // → { readiness, strain, sleep, hrv }
trends.getChart({ metric, days })  // → DataPoint[]
```

---

## 5. Background Jobs

| Job | Trigger | Frequency | Tool |
|-----|---------|-----------|------|
| Garmin sync | Webhook + cron | Real‑time + daily 4 AM | Vercel Cron / Inngest |
| Readiness compute | After sync completes | Daily | Event‑driven |
| Weekly plan generation | Sunday night or on demand | Weekly | Cron |
| Daily modulation | After readiness computed | Daily | Event‑driven |
| Baseline recalculation | After new data | Daily | Event‑driven |
| Anomaly detection | After readiness computed | Daily | Event‑driven |

### Job Queue

```
Garmin webhook received
  → Normalize & store data
  → Compute readiness score
  → Modulate today's workout
  → Send push notification (if significant change)
```

---

## 6. Data Flow

```
1. User syncs Garmin watch → Garmin Connect
2. Garmin Connect → Webhook → Our ingestion service
3. Ingestion service normalizes → PostgreSQL (daily_aggregates, activities)
4. Readiness engine reads data → Computes score → Stores readiness_scores
5. Coaching service reads readiness + profile → Generates/adjusts workout
6. Client app fetches today's readiness + workout via tRPC
7. User sees score + workout recommendation
8. User trains → Garmin records → Back to step 1
```

---

## 7. Security & Privacy

### 7.1 Data Protection

- All Garmin tokens encrypted at rest (AES‑256)
- HTTPS everywhere (TLS 1.3)
- Database connections via SSL
- Row‑level security: users can only access their own data

### 7.2 Auth Flow

```
1. User signs up (Discord OAuth via Better-Auth)
2. User connects Garmin (OAuth 1.0a)
3. Garmin tokens stored encrypted in DB
4. Session managed via Better-Auth (database sessions + cookies)
5. tRPC procedures validate session on every request
```

### 7.3 Privacy

- Clear consent screen before Garmin connection
- Data export (GDPR: right to portability)
- Account deletion removes all data (GDPR: right to erasure)
- No data sold or shared with third parties
- Transparent scoring: user can see exactly how readiness is computed

---

## 8. Deployment

### 8.1 Environments

| Environment | Purpose | URL |
|-------------|---------|-----|
| Development | Local dev | localhost:3000 |
| Preview | PR previews | pr‑{n}.vercel.app |
| Staging | Pre‑production | staging.garmincoach.app |
| Production | Live | garmincoach.app |

### 8.2 Infrastructure as Code

```
infra/
├── docker-compose.yml    # Local dev (Postgres + Redis)
├── vercel.json           # Vercel deployment config
├── drizzle.config.ts     # Database migrations
└── github/
    └── workflows/
        ├── ci.yml        # Lint + test + type check
        ├── preview.yml   # Deploy PR previews
        └── deploy.yml    # Production deployment
```

---

## 9. Monitoring & Observability

| Concern | Tool | What |
|---------|------|------|
| Errors | Sentry | Unhandled exceptions, API errors |
| Analytics | PostHog | User events, funnel analysis |
| Uptime | Vercel / BetterStack | Health checks, alerts |
| Logs | Vercel Logs / Axiom | Structured JSON logs |
| Performance | Vercel Analytics | Core Web Vitals, API latency |

---

## 10. Mobile Architecture (Expo)

```
mobile/
├── app/                  # Expo Router (file‑based)
│   ├── (tabs)/
│   │   ├── index.tsx     # Home / Today
│   │   ├── trends.tsx    # History
│   │   ├── chat.tsx      # Coach chat
│   │   └── settings.tsx  # Profile
│   ├── workout/[id].tsx  # Workout detail
│   └── onboarding/       # Setup flow
├── components/           # Shared UI components
├── lib/
│   ├── api.ts           # tRPC client
│   ├── storage.ts       # Offline cache (MMKV)
│   └── notifications.ts # Push notifications
└── assets/
```

### Offline Support

- Cache last 3 days of readiness + workouts in MMKV
- Show cached data when offline with "Last updated" indicator
- Queue chat messages for send when back online
