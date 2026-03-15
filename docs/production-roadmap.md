# Production Roadmap — GarminCoach

Complete, implementation‑focused roadmap using **T3 Turbo** monorepo to build,
test, and publish the WHOOP‑like Garmin coaching MVP as both a website (Vercel)
and Android app (Google Play).

---

## Tech Stack Decision: T3 Turbo Monorepo

Uses the official T3 Turbo template — a single TypeScript codebase with:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Web | Next.js 15 (App Router) | Website at garmincoach.app |
| Mobile | Expo SDK 51+ / React Native | Native Android app |
| API | tRPC v11 | End‑to‑end typesafe API |
| ORM | Drizzle ORM | Type-safe, SQL-like DB access + migrations |
| Database | PostgreSQL (Neon / Supabase) | Relational + JSONB |
| Build | Turborepo | Fast parallel monorepo builds |
| Styling | Tailwind + NativeWind | Shared styling web + mobile |
| Testing | Vitest + Playwright + Maestro | Unit / integration / E2E |

---

## Phase 0: Prerequisites & Accounts (Day 1)

### 0.1 — Garmin API Access (Start Immediately — 2 Business Day Wait)

Submit the Garmin Health API developer form:
https://www.garmin.com/en-US/forms/GarminConnectDeveloperAccess/

Describe the app:
> "MVP coaching app that ingests Health + Activity data to compute a daily
> Readiness score (0–100) and generate sport/goal‑specific workouts."

Approval is typically **2 business days** (free, works for indie apps).

**Alternative (ship same day):** Use [Terra API](https://tryterra.co) — normalized
Garmin data + webhooks instantly, 100k free credits/month. Swap to direct Garmin later.

### 0.2 — Required Accounts

| Account | Cost | Purpose |
|---------|------|---------|
| GitHub | Free | Source control, CI/CD |
| Vercel | Free tier | Web deployment |
| Expo (EAS) | Free tier | Android cloud builds |
| Neon or Supabase | Free tier | Managed PostgreSQL |
| Google Play Console | $25 one‑time | Android app publishing |
| OpenAI | Pay‑as‑you‑go | AI chat (optional, add later) |

### 0.3 — Local Machine Setup (Fedora 41 / Ubuntu / macOS)

See [Dev Environment Guide](dev-environment.md) for full Fedora 41 instructions.

```bash
# Node.js 20+ (via system package manager or volta)
sudo dnf install nodejs          # Fedora
# or: volta install node@20      # Any OS

# Essential global tools
npm install -g pnpm eas-cli

# Android Studio (for emulator)
sudo dnf install android-studio  # Fedora
# or: download from developer.android.com

# Database (local dev)
sudo dnf install podman docker-compose  # or Docker Desktop
```

---

## Phase 1: Monorepo Bootstrap (Day 1–2)

### 1.1 — Initialize T3 Turbo

```bash
npx create-turbo@latest garmin-coach \
  -e https://github.com/t3-oss/create-t3-turbo
cd garmin-coach
pnpm install
```

### 1.2 — Project Structure

```
garmin-coach/
├── apps/
│   ├── nextjs/               # Next.js 15 website
│   └── expo/                 # Expo / React Native Android app
├── packages/
│   ├── api/                  # tRPC routers (shared web + mobile)
│   ├── auth/                 # Better-Auth configuration
│   ├── db/                   # Drizzle schema + client
│   ├── engine/               # Readiness + strain + coaching logic
│   ├── garmin/               # Garmin API integration utilities
│   ├── ui/                   # Shared UI components
│   └── validators/           # Zod schemas (shared validation)
├── tooling/
│   ├── eslint/               # Shared ESLint config
│   ├── prettier/             # Shared Prettier config
│   └── typescript/           # Shared tsconfig
├── docker-compose.yml        # Local Postgres + Redis
├── turbo.json
└── package.json
```

### 1.3 — Local Dev Environment

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: garmincoach
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

```bash
docker compose up -d
echo 'DATABASE_URL="postgresql://dev:dev@localhost:5432/garmincoach"' > .env
```

### 1.4 — CI Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test
      - run: pnpm build
```

### 1.5 — Tooling Setup

- [ ] Configure ESLint + Prettier + Husky pre‑commit hooks
- [ ] Set up Vitest for unit/integration tests
- [ ] Set up Playwright for web E2E tests
- [ ] Configure Sentry error tracking
- [ ] Verify `pnpm dev` starts both web + expo

**Phase 1 deliverable:** Running monorepo with CI, local DB, both apps starting.

---

## Phase 2: Database Schema & Auth (Days 2–3)

### 2.1 — Drizzle Schema

```typescript
// packages/db/src/schema.ts
import { relations, sql } from "drizzle-orm";
import { pgTable } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod/v4";

// Auth tables (user, session, account, verification) are managed
// by Better-Auth via packages/db/src/auth-schema.ts

// --- User Profile (extends auth user) ---
export const Profile = pgTable("profile", (t) => ({
  id: t.uuid().notNull().primaryKey().defaultRandom(),
  userId: t.text().notNull().unique(),
  age: t.integer(),
  sex: t.varchar({ length: 20 }),
  massKg: t.doublePrecision(),
  heightCm: t.doublePrecision(),
  timezone: t.varchar({ length: 50 }).default("UTC"),
  experienceLevel: t.varchar({ length: 20 }).default("intermediate"),
  primarySports: t.jsonb().$type<string[]>().default([]),
  goals: t.jsonb().$type<{ sport: string; goalType: string; target?: string }[]>().default([]),
  weeklyDays: t.jsonb().$type<string[]>().default([]),
  minutesPerDay: t.integer().default(45),
  maxHr: t.integer(),
  restingHrBaseline: t.doublePrecision(),
  hrvBaseline: t.doublePrecision(),
  sleepBaseline: t.doublePrecision(),
  createdAt: t.timestamp().defaultNow().notNull(),
  updatedAt: t.timestamp({ mode: "date", withTimezone: true }).$onUpdateFn(() => sql`now()`),
}));

export const CreateProfileSchema = createInsertSchema(Profile, {
  sex: z.enum(["male", "female", "other"]).optional(),
  experienceLevel: z.enum(["beginner", "intermediate", "advanced"]).optional(),
}).omit({ id: true, createdAt: true, updatedAt: true });

// --- Daily Metrics (Garmin health data) ---
export const DailyMetric = pgTable("daily_metric", (t) => ({
  id: t.uuid().notNull().primaryKey().defaultRandom(),
  userId: t.text().notNull(),
  date: t.date().notNull(),
  sleepScore: t.integer(),
  totalSleepMinutes: t.integer(),
  deepSleepMinutes: t.integer(),
  remSleepMinutes: t.integer(),
  lightSleepMinutes: t.integer(),
  awakeMinutes: t.integer(),
  hrv: t.doublePrecision(),
  restingHr: t.integer(),
  maxHr: t.integer(),
  stressScore: t.integer(),
  bodyBatteryStart: t.integer(),
  bodyBatteryEnd: t.integer(),
  steps: t.integer(),
  calories: t.integer(),
  garminTrainingReadiness: t.integer(),
  garminTrainingLoad: t.doublePrecision(),
  rawGarminData: t.jsonb(),
  syncedAt: t.timestamp().defaultNow().notNull(),
})); // + unique constraint on (userId, date)

// --- Activity, ReadinessScore, WeeklyPlan, DailyWorkout, ChatMessage ---
// (similar pgTable definitions — see packages/db/src/schema.ts for full schema)
```

### 2.2 — Seed Script

```bash
# packages/db/src/seed.ts
# Generate 1 mock user + 30 days of realistic fake Garmin data
# (HRV: 35–65ms, resting HR: 55–70, sleep: 300–520 min, etc.)
pnpm db:push
```

### 2.3 — Authentication

- Configure Better-Auth with Discord OAuth provider
- Add Expo support via @better-auth/expo plugin
- Create tRPC auth middleware (`protectedProcedure`)
- Write tests: signup via Discord OAuth, session validation, protected routes

**Phase 2 deliverable:** Full schema deployed, seed data, auth working.

---

## Phase 3: Garmin Integration & Data Pipeline (Days 3–6)

### 3.1 — Garmin OAuth Flow

```typescript
// packages/garmin/src/oauth.ts
// Garmin uses OAuth 1.0a (NOT 2.0) — use `oauth-1.0a` npm package
//
// Flow:
// 1. Get request token from Garmin
// 2. Redirect user to Garmin authorization page
// 3. Handle callback with verifier
// 4. Exchange for access token
// 5. Store encrypted tokens in GarminTokens table
```

For Expo mobile: use `expo-web-browser` + `expo-auth-session` (PKCE flow).

### 3.2 — Webhook Receiver

```typescript
// apps/nextjs/src/app/api/garmin/webhook/route.ts
// Public endpoint — Garmin pushes data here
//
// 1. Verify signature (Garmin provides header)
// 2. Parse payload type (dailies, activities, sleep, stress)
// 3. Normalize Garmin JSON → our DailyMetric / Activity schema
// 4. Upsert into database (idempotent on userId + date)
// 5. Trigger readiness recomputation
```

### 3.3 — Backfill (Onboarding)

```typescript
// packages/garmin/src/backfill.ts
// On "Connect Garmin" success:
// 1. Pull Daily Summaries for last 30 days
// 2. Pull Activity list for last 30 days
// 3. Normalize and bulk‑insert
// 4. Compute baselines from imported data
// 5. Compute readiness for today
```

### 3.4 — Normalizer

```typescript
// packages/garmin/src/normalize.ts
// Maps Garmin API JSON fields → our Drizzle schema
//
// Key mappings:
//   sleepDurationInSeconds ÷ 60 → totalSleepMinutes
//   restingHeartRateInBeatsPerMinute → restingHr
//   averageStressLevel → stressScore
//   bodyBatteryChargedValue → bodyBatteryStart
//   activityType → sportType
//   vO2MaxValue → vo2maxEstimate
//
// Handle missing fields: set null, don't crash
```

### 3.5 — Tests

- [ ] OAuth round‑trip (mock Garmin responses)
- [ ] Webhook signature verification
- [ ] Normalizer: all field mappings, missing fields, edge cases
- [ ] Backfill: 30‑day import, deduplication
- [ ] Sync idempotency

**Phase 3 deliverable:** User connects Garmin → sees 30 days of data in DB.

---

## Phase 4: Readiness & Strain Engine (Days 6–8)

### 4.1 — Core Engine (`packages/engine/`)

```
packages/engine/src/
├── readiness/
│   └── index.ts              # All 6 component scorers + calculateReadiness()
├── strain/
│   └── index.ts              # TRIMP, strain score, ACWR, consecutive hard days
├── baselines/
│   └── index.ts              # EMA, population defaults, personal baselines
├── anomalies/
│   └── index.ts              # HRV crash, RHR spike, sleep deficit, overreaching
├── coaching/
│   ├── index.ts              # Weekly planner, modulation, daily generation
│   └── templates/
│       └── index.ts          # 20+ workout templates (running, cycling, strength)
├── types.ts                  # Shared type definitions
└── index.ts                  # Public API exports
```

### 4.2 — Readiness Function Signature

```typescript
// packages/engine/src/readiness/index.ts
export function calculateReadiness(input: {
  todayMetrics: DailyMetricInput;
  recentStrainScores: number[];  // most recent first, last 7 days
  baselines: Baselines;          // { hrv, restingHr, sleep, dailyStrainCapacity }
}): ReadinessResult {
  // Returns: { score, zone, color, explanation, components }
}
```

See [Readiness Engine](readiness-engine.md) for full algorithm pseudocode.

### 4.3 — Unit Tests (100% Coverage Target)

```typescript
// packages/engine/src/readiness/__tests__/readiness.test.ts
describe('calculateReadiness', () => {
  test('Prime day when all metrics above baseline', () => { ... });
  test('Poor day when sleep < 5h and HRV crashed', () => { ... });
  test('Handles missing HRV gracefully (device without HRV)', () => { ... });
  test('Cold start: uses population defaults for day 1', () => { ... });
  test('Blends personal/default baselines for days 1–30', () => { ... });
  test('TRIMP calculation matches manual formula', () => { ... });
  test('ACWR flags overreaching above 1.5', () => { ... });
  test('Anomaly detection: HRV crash for 2+ consecutive days', () => { ... });
});
```

**Phase 4 deliverable:** Fully tested readiness + strain engine, accessible as a package.

---

## Phase 5: Coaching Logic & Workout Generation (Days 8–11)

### 5.1 — Workout Template Library

Hard‑code 20–30 templates in `packages/engine/src/coaching/templates/`:

```typescript
// packages/engine/src/coaching/templates/index.ts
export interface WorkoutTemplate {
  id: string;
  sport: string;
  workoutType: string;
  title: string;
  description: string;
  intensity: "easy" | "moderate" | "hard" | "very_hard";
  durationRange: [number, number];
  hrZoneRange: [number, number];
  strainRange: [number, number];
  structure: WorkoutStructureBlock[];
}

// Example:
export const runningTemplates: WorkoutTemplate[] = [
  {
    id: "run-easy",
    sport: "running",
    workoutType: "easy_run",
    title: "Easy Run",
    description: "Continuous easy pace, conversational effort",
    intensity: "easy",
    durationRange: [30, 45],
    hrZoneRange: [2, 2],
    strainRange: [4, 7],
    structure: [
      { phase: "warmup", description: "Walk 2 min, then easy jog", durationMinutes: 5, hrZone: 1 },
      { phase: "main", description: "Easy pace run", durationMinutes: 25, hrZone: 2 },
      { phase: "cooldown", description: "Walk 3-5 min", durationMinutes: 5, hrZone: 1 },
    ],
  },
  // ... 20+ templates for running, cycling, and strength
];
```

Same for cycling, strength, swimming templates.

### 5.2 — Weekly Plan Generator

```typescript
// packages/engine/src/coaching/index.ts
export function selectWeeklyTemplate(
  sport: string,
  goalType: string,
  availableDays: number,
): WeekSlot[]

export function generateDailyWorkout(
  sport: string,
  goalType: string,
  dayOfWeek: number,    // 0 = Monday
  availableDays: number,
  readinessZone: ReadinessZone,
  recentHardDays: number,
): WorkoutRecommendation
```

- Template selection matrix: sport × goal × days/week
- Day assignment: hardest sessions separated, long on weekends
- Volume progression: +5–10%/week cap, deload every 4th week

### 5.3 — Daily Readiness Modulation

```typescript
// packages/engine/src/coaching/index.ts
export function modulateWorkout(
  template: WorkoutTemplate,
  readinessZone: ReadinessZone,
  sport: string,
): WorkoutRecommendation
```

- Prime → promote hard session or intensify +5%
- High → execute as planned
- Moderate → reduce volume −10%
- Low → substitute easy/technique, reduce −20%
- Poor → rest or 20min Zone 1 only
- Hard day stacking prevention: force easy after 2+ consecutive hard days

### 5.4 — Parameterization from History

- Pace targets from recent best efforts or VO2max (Daniels' tables)
- Power targets for cycling (% FTP)
- Strength: auto‑progress weight when all reps completed, reduce on low readiness

### 5.5 — Tests

- [ ] Template selection for every sport × goal × days combination
- [ ] Weekly plan day assignment logic
- [ ] Readiness modulation for every zone
- [ ] Hard day stacking prevention
- [ ] Volume progression caps
- [ ] Pace estimation from VO2max

**Phase 5 deliverable:** Full coaching engine generating personalized daily workouts.

---

## Phase 6: tRPC API Layer (Days 11–13)

### 6.1 — Router Structure

```typescript
// packages/api/src/root.ts
export const appRouter = createTRPCRouter({
  auth:      authRouter,
  garmin:    garminRouter,
  post:      postRouter,
  profile:   profileRouter,
  readiness: readinessRouter,
  workout:   workoutRouter,
  trends:    trendsRouter,
});
```

### 6.2 — Key Procedures

```typescript
// Garmin
garmin.getConnectionStatus()     // → { connected, lastSync }
garmin.initiateOAuth()           // → { authUrl }
garmin.handleCallback({ code })  // → store tokens
garmin.triggerBackfill({ days }) // → { metricsInserted, activitiesInserted }

// Readiness
readiness.getToday()             // → { score, zone, explanation, factors }
readiness.getHistory({ days })   // → ReadinessScore[]
readiness.getComponents({ date })// → { sleep, hrv, rhr, load, stress }
readiness.getAnomalies()         // → AnomalyAlert[]

// Workout
workout.getToday()               // → DailyWorkout
workout.adjustDifficulty({ direction }) // → WorkoutRecommendation
workout.getWeekPlan()            // → DailyWorkout[]
workout.getDetail({ id })        // → { structure, targets, explanation }

// Chat — Planned — not yet implemented

// Trends
trends.getSummary({ period })    // → { readiness, strain, sleep, hrv }
trends.getChart({ metric, days })// → DataPoint[]
```

### 6.3 — Integration Tests

Test every procedure with a test DB:

```typescript
describe('readiness.getToday', () => {
  test('returns computed score for authenticated user', async () => {
    // seed user + metrics → call procedure → verify response shape + values
  });
  test('returns null gracefully for user with no data', async () => { ... });
  test('rejects unauthenticated requests', async () => { ... });
});
```

**Phase 6 deliverable:** Complete API layer with integration tests.

---

## Phase 7: Frontend — Web App (Days 13–18)

### 7.1 — Onboarding Flow (3 screens)

1. **Step 1: About You** — Age, sex, weight, height
2. **Step 2: Sports & Goals** — Multi-select sports, goal per sport
3. **Step 3: Weekly Schedule** — Day toggles, minutes-per-session slider, confirmation

### 7.2 — Home / Today Screen

- Large readiness score circle with zone color
- One‑sentence insight text
- Today's workout card (title, duration, zone)
- "Too tired" / "Feeling fresh" adjustment buttons
- Quick stats: sleep, HRV, steps, strain

### 7.3 — Workout Detail Screen

- Structured blocks: warm‑up → main set → cool‑down
- Target HR zones, pace, duration
- "Why this today" explanation
- "Start Workout" CTA

### 7.4 — History / Trends Screen

- 7‑day and 28‑day toggle
- Readiness vs strain dual‑axis chart (Recharts)
- Sleep and HRV trend lines
- Weekly summary stats with ▲/▼ vs prior week

### 7.5 — Coach Chat Screen

- Chat bubbles with streaming responses
- Quick action chips for common questions
- Chat history

### 7.6 — Settings

- Edit profile, manage sports/goals
- Garmin connection status
- Data export, account deletion
- Privacy policy link

### 7.7 — Component Library (`packages/ui/`)

Shared components using shadcn/ui + Tailwind:

```
packages/ui/src/
├── readiness-card.tsx
├── workout-card.tsx
├── stat-row.tsx
├── zone-badge.tsx
├── chart/
│   ├── readiness-chart.tsx
│   └── trend-line.tsx
└── layout/
    ├── bottom-nav.tsx
    └── screen-wrapper.tsx
```

### 7.8 — Web E2E Tests (Playwright)

```typescript
// apps/nextjs/e2e/onboarding.spec.ts
test('complete onboarding flow', async ({ page }) => {
  await page.goto('/onboarding');
  // Fill profile → select sport → set availability → verify home screen
});

// apps/nextjs/e2e/daily-flow.spec.ts
test('view readiness and workout', async ({ page }) => {
  // Login → see readiness card → click workout → see detail
});
```

**Phase 7 deliverable:** Fully functional web app with E2E tests.

---

## Phase 8: AI Coach Chat (Days 18–19)

### 8.1 — Chat Backend

```typescript
// packages/api/src/routers/chat.ts
// Uses Vercel AI SDK + OpenAI GPT‑4o‑mini
//
// System prompt:
// "You are a deterministic fitness coach. Only use the user's stored
//  DailyMetric, Profile, and Readiness data (provided in context).
//  Never give medical advice. Acknowledge uncertainty."
//
// Context injection: latest readiness, last 7 days metrics,
// current workout plan, profile/goals
```

### 8.2 — Supported Intents

| Intent | Trigger Phrases | Handler |
|--------|----------------|---------|
| Daily recommendation | "What should I do today?" | Return today's workout + explanation |
| Readiness explanation | "Why is my readiness low/high?" | Top factors from computation |
| Adjust difficulty | "Make it harder/easier" | Re‑generate ±1 zone shift |
| Race taper | "I have a race in N days" | Mini‑taper plan generation |
| Weekly summary | "How was my week?" | Aggregate stats + insights |

### 8.3 — Guardrails

- Only reference user's own stored metrics
- Never provide medical advice
- Prefix uncertain statements: "Based on your data, I'd suggest…"
- Escalation: "Consider consulting a coach or doctor if…"

### 8.4 — Tests

- [ ] Intent classification accuracy
- [ ] Guardrail enforcement (inject medical questions → verify refusal)
- [ ] Context injection (verify metrics appear in LLM context)
- [ ] Streaming response handling

**Phase 8 deliverable:** Working coach chat with data‑backed answers.

---

## Phase 9: Mobile App — Expo (Days 19–23)

### 9.1 — Expo Setup

```bash
cd apps/expo
npx expo install expo-router expo-web-browser expo-auth-session
npx expo install @react-native-mmkv/storage  # offline cache
npx expo install react-native-reanimated      # animations
```

### 9.2 — Screen Structure (Expo Router)

```
apps/expo/app/
├── (tabs)/
│   ├── index.tsx         # Home / Today
│   ├── trends.tsx        # History
│   ├── chat.tsx          # Coach chat
│   └── settings.tsx      # Profile
├── workout/[id].tsx      # Workout detail
├── onboarding/
│   ├── index.tsx         # Welcome
│   ├── profile.tsx       # Profile setup
│   ├── sports.tsx        # Sport & goal
│   └── availability.tsx  # Weekly schedule
└── _layout.tsx           # Root layout
```

### 9.3 — Shared Code with Web

- tRPC client configured for mobile (`packages/api`)
- Shared validators (`packages/validators`)
- Shared engine logic (`packages/engine`)
- Charts: `react-native-gifted-charts` or Victory Native

### 9.4 — Offline Support

```typescript
// apps/expo/lib/storage.ts
// Cache last 3 days of readiness + workouts in MMKV
// Show cached data when offline with "Last updated X ago" indicator
// Queue chat messages for send when back online
```

### 9.5 — Push Notifications

- Morning readiness notification (6–7 AM local)
- Workout reminder
- Anomaly alerts (HRV crash, sleep deficit)

### 9.6 — Mobile E2E Tests (Maestro)

```yaml
# e2e/maestro/daily-flow.yaml
appId: com.garmincoach.app
---
- launchApp
- assertVisible: "Readiness"
- tapOn: "View Details"
- assertVisible: "Warm-up"
- tapOn: "Start Workout"
```

**Phase 9 deliverable:** Android app running on emulator + device with all features.

---

## Phase 10: Testing & Quality (Days 23–26)

### 10.1 — Test Coverage Targets

| Layer | Tool | Target | Scope |
|-------|------|--------|-------|
| Unit | Vitest | ≥90% | Engine, utilities, validators |
| Integration | Vitest + test DB | ≥80% | All tRPC procedures |
| E2E Web | Playwright | All flows | Onboarding, daily, chat, trends |
| E2E Mobile | Maestro | Core flows | Onboarding, daily, workout |
| API | Vitest + supertest | 100% | Endpoint contracts |
| Load | k6 | Pass | 100 concurrent users |

### 10.2 — Test File Convention

```
packages/engine/src/
├── readiness/
│   ├── index.ts
│   ├── index.test.ts                    ← Unit
│   └── index.integration.test.ts        ← Integration
packages/api/src/routers/
├── readiness.ts
├── readiness.test.ts                    ← Router test
apps/nextjs/e2e/
├── onboarding.spec.ts                   ← E2E
├── daily-flow.spec.ts
└── chat.spec.ts
```

### 10.3 — Performance Benchmarks

- API response time: < 200ms (p95)
- Mobile cold start: < 2s
- Readiness computation: < 50ms
- Web Lighthouse: ≥ 90 performance score

### 10.4 — Security Checklist

- [ ] Auth tokens: httpOnly cookies, short‑lived JWTs
- [ ] Garmin tokens: encrypted at rest (AES‑256)
- [ ] Row‑level security: users can only access their own data
- [ ] HTTPS everywhere (TLS 1.3)
- [ ] Rate limiting on all public endpoints
- [ ] Input validation on all mutations (Zod)
- [ ] CORS configured for known origins only

**Phase 10 deliverable:** All tests passing, security hardened.

---

## Phase 11: Deployment & Publishing (Days 26–28)

### 11.1 — Website → Vercel

```bash
cd apps/nextjs
vercel deploy --prod
```

- Custom domain: garmincoach.app
- Environment variables: DATABASE_URL, GARMIN_*, OPENAI_KEY, AUTH_SECRET, AUTH_DISCORD_ID, AUTH_DISCORD_SECRET
- Preview deploys on every PR

### 11.2 — Android App → Google Play

```bash
# Build production AAB
eas build --platform android --profile production

# Submit to Google Play
eas submit --platform android
```

**Google Play requirements:**
- App icon (512×512) + feature graphic (1024×500)
- 4–8 screenshots
- Privacy policy URL (host on Carrd or your website)
- Content rating questionnaire
- Data safety form (declare Garmin health data usage)

**Timeline:** Internal testing → Closed beta (optional) → Production
Google Play review: typically 3–7 days for health/fitness apps.

### 11.3 — Infrastructure Checklist

- [ ] Sentry error tracking in all environments
- [ ] PostHog analytics for key user actions
- [ ] Uptime monitoring (BetterStack or Vercel)
- [ ] Database backups enabled (Neon/Supabase auto‑backup)
- [ ] Structured JSON logging

### 11.4 — Documentation

- [ ] API docs (auto‑generated from tRPC types)
- [ ] Privacy policy & terms of service
- [ ] User help / FAQ page
- [ ] Garmin data consent explanation page

**Phase 11 deliverable:** Live website + Android app on Play Store.

---

## Phase 12: Launch & "Done" Verification (Days 28–30)

### 12.1 — MVP "Done" Checklist

A single Garmin user can:

- [x] Connect their Garmin account → see 30 days of backfilled data
- [x] See a daily readiness score (0–100) with color zone and one‑line explanation
- [x] Get one main workout recommendation per day that:
  - Matches their chosen sport and goal
  - Adjusts volume and intensity based on readiness
- [x] Ask basic questions ("why low?", "harder/easier?") → data‑backed answers
- [x] Access via website (garmincoach.app) or Android app (Play Store)

### 12.2 — All Tests Green

```bash
pnpm test          # Unit + integration
pnpm test:e2e      # Playwright web E2E
pnpm test:mobile   # Maestro mobile E2E
pnpm lint          # ESLint
pnpm type-check    # TypeScript
```

---

## Post‑MVP Roadmap

### M1 — Training API Integration (1 week post‑launch)

- Push structured workouts to Garmin watch
- "Export to Garmin" button on workout detail
- Garmin Training & Courses API integration

### M2 — Periodization & Polish (2 weeks)

- Multi‑week periodization view (mesocycle planning)
- Enhanced AI chat (follow‑up questions, conversation context)
- iOS app (EAS builds for iOS, submit to App Store)

### M3 — Scale & Monetize (Month 2+)

- Subscription plans (free tier + premium)
- Team/coach portal
- Advanced analytics (training peaks, performance predictions)
- Wearable integrations beyond Garmin (Apple Watch, Polar, COROS)

---

## Dependency Graph

```
Phase 0 (Prerequisites)
 └── Phase 1 (Monorepo Bootstrap)
      ├── Phase 2 (Schema & Auth)
      │    └── Phase 3 (Garmin Integration)
      │         └── Phase 4 (Readiness Engine)
      │              └── Phase 5 (Coaching Logic)
      │                   └── Phase 6 (tRPC API)
      │                        ├── Phase 7 (Web App)        ─┐
      │                        ├── Phase 8 (AI Chat)         ├── Phase 10 (Testing)
      │                        └── Phase 9 (Mobile App)     ─┘       │
      │                                                          Phase 11 (Deploy)
      │                                                               │
      │                                                          Phase 12 (Launch)
```

**Phases 7, 8, and 9 can run in parallel** once Phase 6 is complete.

---

## Effort Summary

| Phase | Duration | Focus |
|-------|----------|-------|
| 0: Prerequisites | Day 1 | Accounts, Garmin form, tooling |
| 1: Bootstrap | Days 1–2 | Monorepo, CI, local dev |
| 2: Schema & Auth | Days 2–3 | Drizzle, Better-Auth |
| 3: Garmin | Days 3–6 | OAuth, webhooks, backfill, normalizer |
| 4: Readiness Engine | Days 6–8 | Score calculation, baselines, anomalies |
| 5: Coaching Logic | Days 8–11 | Templates, planner, modulation |
| 6: tRPC API | Days 11–13 | All routers, integration tests |
| 7: Web App | Days 13–18 | All screens, E2E tests |
| 8: AI Chat | Days 18–19 | LLM integration, guardrails |
| 9: Mobile App | Days 19–23 | Expo screens, offline, notifications |
| 10: Testing | Days 23–26 | Coverage, security, performance |
| 11: Deployment | Days 26–28 | Vercel, Play Store, monitoring |
| 12: Launch | Days 28–30 | Verification, docs |

**Total: ~30 working days (6 weeks part‑time, 3 weeks full‑time)**
**Cost: ~$0 on free tiers until you scale**
