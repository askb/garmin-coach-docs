# Product Specification v2.1 — GarminCoach Sport Scientist

## 1. Product Vision

GarminCoach v2.1 is an evidence-based sport scientist platform that transforms
Garmin wearable data into actionable training intelligence. Every algorithm cites
peer-reviewed research. The system serves athletes from beginner to elite across
running, cycling, strength, swimming, and team sports.

### Core Capabilities

1. **Readiness scoring** — Z-score based daily readiness (0–100) with 6 components,
   confidence levels, and 5 zones (Prime/Good/Moderate/Low/Poor)
2. **Training load management** — TRIMP/strain scoring, ACWR monitoring, CTL/ATL/TSB
   fitness-fatigue modeling, load focus classification
3. **Coaching** — 16 workout templates, weekly planning, readiness-based modulation,
   difficulty adjustment
4. **Analytics** — Trend analysis, correlations, notable change detection, running
   form analysis
5. **Performance estimation** — VO2max (3 methods), race prediction (5K–marathon),
   training status classification
6. **Recovery intelligence** — Recovery time estimation, sleep coaching, anomaly detection
7. **Zone analytics** — HR zone distribution tracking, Seiler's polarization index,
   zone trends, aerobic efficiency trend, activity calendar heatmap, peak performances
8. **AI specialist coaching** — 4 AI agents (Sport Scientist, Psychologist, Nutritionist,
   Recovery Specialist) powered by local Ollama inference with real-time data context

---

## 2. Audience Levels

| Level | Description | System Behavior |
|-------|-------------|-----------------|
| **Beginner** | < 1 year training, learning fundamentals | Conservative load ramps, simpler workouts, more recovery |
| **Intermediate** | 1–3 years, established base | Standard progressions, full template library |
| **Advanced** | 3+ years, performance-oriented | Higher load tolerance, advanced periodization, race-specific |
| **Elite** | Competitive athlete | Full feature set, aggressive optimization, peaking protocols |

---

## 3. Data Layer

### 3.1 Data Sources (Garmin Health & Activity APIs)

| Source | Fields Ingested |
|--------|----------------|
| Daily health summaries | Steps, calories, distance, stress, sleep duration/stages, resting HR, HRV, Body Battery |
| Activity records | Sport type, start/end, distance, duration, pace, HR traces, intensity, VO2max, running dynamics |
| Existing Garmin metrics | Training Load, Recovery Time, Training Readiness (device-dependent) |

### 3.2 Schema (13 Tables — Drizzle + PostgreSQL)

| Table | Key Fields | Purpose |
|-------|-----------|---------|
| Profile | age, sex, mass, height, experience, sports, goals, baselines | User configuration |
| DailyMetric | 30+ fields: sleep stages, HRV, RHR, stress, Body Battery, steps | Daily health data |
| Activity | 25+ fields: sport, duration, distance, HR, pace, TRIMP, strain, VO2max, running dynamics | Activity records |
| ReadinessScore | score, zone, 6 component scores, confidence, explanation | Daily readiness |
| WeeklyPlan | sport, goal, template, week structure | Weekly training plan |
| DailyWorkout | type, title, structure, targets, strain range, status | Individual workouts |
| ChatMessage | role, content, context | AI coach conversations |
| VO2maxEstimate | method, value, confidence, date | VO2max tracking |
| TrainingStatus | status, metrics snapshot, date | Training status history |
| JournalEntry | content, mood, tags, date | Athlete journal |
| CorrelationResult | metric pair, r-value, p-value, period | Statistical correlations |
| RacePrediction | distance, predicted time, method, input race | Race time predictions |
| WorkoutTimeSeries | timestamp, HR, pace, cadence, power | Activity time-series data |

See [Data Model](data-model.md) for complete field-level documentation.

---

## 4. Engine Modules (packages/engine — 131 tests)

### 4.1 Readiness Scoring

**Method:** Z-score based composite scoring (Buchheit 2014)

| Component | Weight | Source | Scoring Method |
|-----------|--------|--------|---------------|
| Sleep quantity | 20% | Garmin sleep | Ratio vs personal baseline |
| Sleep quality | 15% | Garmin sleep score / deep+REM ratio | Quality index |
| HRV vs baseline | 25% | Garmin HRV | Z-score vs 14-day rolling EMA |
| Resting HR vs baseline | 10% | Garmin daily summary | Deviation from 14-day EMA |
| Training load (ACWR) | 20% | Computed strain | ACWR position in sweet spot |
| Stress | 10% | Garmin stress + Body Battery | Inverted composite |

**Zones:**

| Score | Zone | Guidance |
|-------|------|----------|
| 80–100 | **Prime** | High-intensity or long workouts recommended |
| 60–79 | **Good** | Normal training; optional intensity push |
| 40–59 | **Moderate** | Standard or slightly reduced session |
| 20–39 | **Low** | Easy/technique work only |
| 0–19 | **Poor** | Rest or active recovery |

**Confidence levels:** Based on data completeness and baseline maturity.

### 4.2 Strain & TRIMP

**Banister TRIMP (1991):**
```
TRIMP = D × ΔHR_ratio × e^(k × ΔHR_ratio)
  k = 1.92 (male), 1.67 (female)
```

**Strain scoring:** 0–21 scale mapped from TRIMP via exponential saturation curve.

### 4.3 ACWR (Acute:Chronic Workload Ratio)

- **Rolling average:** 7d acute / 28d chronic (Hulin 2016)
- **EWMA variant:** Exponentially weighted, λ_acute = 0.25, λ_chronic = 0.069 (Williams 2017)
- **Risk zones:** < 0.8 (under-prepared), 0.8–1.3 (sweet spot), 1.3–1.5 (caution), > 1.5 (danger)

### 4.4 CTL/ATL/TSB (Fitness-Fatigue Model)

- **CTL** (Chronic Training Load): 42-day EMA — represents fitness
- **ATL** (Acute Training Load): 7-day EMA — represents fatigue
- **TSB** (Training Stress Balance): CTL − ATL — represents form/freshness
- **Ramp rate tracking:** Monitors CTL change rate for overload prevention

### 4.5 Load Focus Classification

Classifies each session and rolling load as:
- **Aerobic** — Predominantly Zone 1–2 work
- **Anaerobic** — High-intensity Zone 4–5 work
- **Mixed** — Balanced distribution across zones

### 4.6 Baselines

- **Method:** 14-day rolling EMA with SD tracking
- **Z-score infrastructure:** Individual deviations from personal baselines
- **Cold start:** Age-adjusted population defaults (Shaffer & Ginsberg 2017) until personal data accumulates
- **Blend:** Weighted transition from population to personal baselines over 14–30 days

### 4.7 Anomaly Detection

| Anomaly | Detection Method | Threshold |
|---------|-----------------|-----------|
| HRV drop | Z-score < −1.5 for 2+ days | Warning (2d) / Critical (3d+) |
| RHR spike | > 5 bpm above baseline for 2+ days | Warning (2d) / Critical (3d+) |
| Sleep disruption | < 6h for 3+ consecutive nights | Critical |
| Overreaching | ACWR > 1.5 + declining HRV | Critical |

### 4.8 VO2max Estimation (3 Methods)

| Method | Input Required | Citation |
|--------|---------------|----------|
| ACSM running equation | Pace + HR data | ACSM 2021 |
| Uth ratio | HRmax + HRrest only | Uth et al. 2004 |
| Cooper test | 12-min run distance | Cooper 1968 |

### 4.9 Race Prediction

- **Riegel formula:** T₂ = T₁ × (D₂/D₁)^1.06
- **VDOT-based:** Daniels' tables for equivalent performances
- **Distances:** 5K, 10K, half marathon, marathon

### 4.10 Training Status Classification

| Status | Criteria | Description |
|--------|----------|-------------|
| **Productive** | VO2max ↑, ACWR 0.8–1.3, HRV stable/↑ | Fitness improving |
| **Maintaining** | VO2max stable, ACWR 0.8–1.3 | Holding fitness |
| **Detraining** | VO2max ↓, ACWR < 0.6 | Insufficient stimulus |
| **Overreaching** | VO2max ↓ despite load, ACWR > 1.5, HRV ↓ | Maladaptive response |
| **Peaking** | ACWR 0.6–0.9, HRV ↑ | Intentional taper |
| **Recovery** | ACWR < 0.8, HRV improving | Post-block recovery |

### 4.11 Recovery Time Estimation

Based on session type with modifiers for:
- Current strain level
- Readiness score
- Age (> 40: 1.1–1.2×, > 55: 1.2–1.4×)
- Sleep debt
- Training age

> Citation: Hausswirth & Mujika 2013

### 4.12 Sleep Coach

- **Sleep need calculation:** Age-adjusted, athlete-adjusted (+1h), strain-adjusted
- **Sleep debt tracking:** Rolling 7-day cumulative deficit
- **Bedtime recommendations:** Based on wake time, sleep need, and sleep onset latency

### 4.13 Trend Analysis

- **Linear regression:** Direction (improving/declining/stable) with significance testing
- **Rolling averages:** 7-day and 28-day smoothed trends
- **Notable change detection:** Flags significant deviations from recent patterns

### 4.14 Correlations

Pearson r with p-values for 6 standard metric pairs:
1. Sleep → Readiness
2. HRV → Readiness
3. Training load → Readiness
4. Sleep → HRV
5. Stress → Readiness
6. RHR → Readiness

### 4.15 Running Form Analysis

| Metric | Elite Benchmark | Analysis |
|--------|----------------|----------|
| Ground contact time (GCT) | 160–200 ms | Efficiency indicator |
| Vertical oscillation | 4–6 cm | Energy waste detection |
| Stride length | Self-selected ± 3% | Overstriding detection |
| Cadence | 170–185 spm | Turnover optimization |
| GCT balance | 50/50 ± 2% | Asymmetry detection |

### 4.16 Coaching Engine

- **16 workout templates** across running, cycling, and strength
- **Weekly planner:** Sport × goal × days/week template selection
- **Readiness-based modulation:** Prime (+5%), Good (as-is), Moderate (−10%), Low (swap to easy), Poor (rest)
- **Hard day stacking prevention:** Force easy after 2+ consecutive hard days

---

## 5. API Layer (tRPC v11 — 39+ endpoints)

### 14 Routers

| Router | Endpoints | Purpose |
|--------|-----------|---------|
| analytics | 7 | Trends, correlations, running form, notable changes, training status |
| auth | 2 | Login, session management |
| chat | 2 | AI specialist agent messaging (Ollama-powered) |
| garmin | 4 | OAuth, webhook, backfill, sync status |
| journal | 4 | CRUD for athlete journal entries |
| post | 4 | Legacy content management |
| profile | 4 | CRUD for user profile and preferences |
| readiness | 4 | Today's score, history, components, anomalies |
| sleep | 3 | Dashboard data, debt tracking, coach recommendations |
| workout | 4 | Today's workout, weekly plan, detail, difficulty adjustment |
| trends | 6 | Summary, multi-metric charts, period comparisons |
| zones | 7 | Zone distribution, polarization index, zone trends, efficiency, calendar, volume, peaks |

---

## 6. Frontend (Next.js 16 + Tailwind + Recharts)

### 6-Tab Navigation

| Tab | Page | Key Features |
|-----|------|-------------|
| Today | Home/Dashboard | Readiness circle, workout card, quick stats, adjustment buttons |
| Trends | Advanced Trends | Multi-metric overlay charts, trend regression, correlations table, notable changes |
| Training | Training Load | CTL/ATL/TSB line chart, ACWR gauge, load focus pie chart, recovery estimation, training status badge |
| Zones | Zone Analytics | Weekly HR zone distribution, polarization index, zone trends, efficiency scatter, activity calendar, volume by sport, peak performances |
| Sleep | Sleep Dashboard | Sleep stages bar chart, sleep score, debt tracker, sleep coach advice, timing analysis |
| Settings | Settings | Profile edit, Garmin connection, preferences |

### Additional Pages

- **Onboarding** — 3-step flow: About You → Sports & Goals → Weekly Schedule
- **Workout Detail** — Structured blocks (warm-up/main/cooldown), target zones, "Why this today" explanation
- **AI Coach** — 4 specialist agent tabs, quick prompt chips, markdown rendering, typing indicator

---

## 7. Testing

| Category | Count | Tool | Scope |
|----------|-------|------|-------|
| Engine unit tests | 131 | Vitest | All engine modules — readiness, strain, ACWR, CTL, baselines, anomalies, VO2max, race prediction, training status, recovery, sleep, trends, correlations, running form, coaching |
| E2E tests | 20 | Playwright | All user flows — onboarding, daily, trends, training, sleep |
| Integration tests | 10 | Vitest | tRPC router contracts |
| Type checking | 16/16 | TypeScript | All turbo tasks passing |
| **Total** | **161** | | |

---

## 8. Infrastructure

- **Monorepo:** T3 Turbo (pnpm + Turborepo)
- **Containers:** Docker (PostgreSQL 16 + Redis 7)
- **CI/CD:** GitHub Actions (lint + typecheck + test + build)
- **Auth:** Better-Auth with Discord OAuth
- **ORM:** Drizzle ORM with drizzle-kit migrations

---

## 9. What's Built vs. Remaining

### ✅ Complete (Phases 1–12, plus v2.1 additions)

- Full engine with 131 tests and evidence-based citations
- 13-table PostgreSQL schema via Drizzle
- 39+ tRPC endpoints across 14 routers (including zones + chat)
- 11+ frontend pages with Recharts visualizations
- Zone Analytics dashboard with 7 chart sections
- AI Specialist Agents (4 personas via local Ollama)
- Onboarding, settings, Garmin OAuth flow
- GitHub Actions CI pipeline
- Docker development environment with resource limits
- Health monitoring script (scripts/health-check.sh)
- 16/16 turbo typecheck tasks passing

### 🔲 Remaining (Phases 13–16)

| Phase | Feature | Description |
|-------|---------|-------------|
| 13 | Activity Detail Page | Individual activity view with time-series charts, splits, HR zones |
| 14 | Journal | Daily journal entries with mood tracking, tags, and correlation to metrics |
| ~~15~~ | ~~AI Coach Chat~~ | ✅ **Completed** — AI specialist agents with local Ollama inference |
| 16 | VO2max & Predictions Page | VO2max history chart, race prediction calculator UI, training paces |
