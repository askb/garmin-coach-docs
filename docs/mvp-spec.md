# MVP Specification — GarminCoach

## 1. Core User Outcomes (MVP Scope)

Your MVP should reliably do three things:

- Turn Garmin data into a simple daily **readiness** score (0–100) and clear color zone.
- Translate readiness into concrete workout targets: duration, intensity, and focus
  (e.g., "easy endurance run 40–50 minutes, HR Zone 2–3").
- Adapt to sport and goal (e.g., "general fitness", "5K PB", "hypertrophy",
  "triathlon base") with minimal config from the user.

Everything else (social, periodization views, advanced AI chat) is "nice later",
not MVP‑critical.

---

## 2. Data Layer: What You Pull From Garmin

Use the [Garmin Health & Activity APIs](https://developer.garmin.com/gc-developer-program/health-api/)
as your backbone.

### 2.1 Data Sources

| Source | Fields |
|--------|--------|
| Daily health summaries | steps, calories, distance, stress, sleep duration/stages, resting HR, HRV, Body Battery |
| Activity records | sport type, start/end, distance, duration, pace, HR traces, intensity, VO2max |
| Existing Garmin metrics | Training Load, Recovery Time, Training Readiness (device‑dependent) |

**MVP requirement:** daily sync job that ingests last 7–30 days, then incremental
updates when the user syncs to Garmin Connect.

### 2.2 Minimal Data Model (per user)

- **Profile:** age, sex, mass, height, timezone, experience level, primary sports, goals.
- **Daily aggregates:** date, sleep score, total sleep, HRV, resting HR, max HR,
  stress score, Body Battery, steps, calories, Garmin Training Readiness if present.
- **Activities:** sport_type, distance, duration, avg HR, max HR, TRIMP/strain score
  (computed), Garmin training load if present.

See [Data Model](data-model.md) for full schema.

---

## 3. Readiness & Strain Engine

See [Readiness Engine](readiness-engine.md) for algorithm details.

### 3.1 Daily Readiness Score (0–100)

Inputs (weighted):

| Factor | Weight | Source |
|--------|--------|--------|
| Sleep quantity vs individual baseline | 20% | Garmin sleep |
| Sleep quality (efficiency, deep/REM) | 15% | Garmin sleep score |
| HRV vs personal 30‑day rolling baseline | 25% | Garmin HRV |
| Resting HR vs baseline | 10% | Garmin daily summary |
| Recent training load (last 3–7 days) | 20% | Computed strain |
| Daytime stress and Body Battery | 10% | Garmin stress |

### Readiness Zones

| Score | Zone | Guidance |
|-------|------|----------|
| 80–100 | **Prime** | High‑intensity or long workouts recommended |
| 60–79 | **High/Good** | Normal training; optional intensity |
| 40–59 | **Moderate** | Standard or slightly reduced session |
| 20–39 | **Low/Strained** | Short/easy or technique work |
| 0–19 | **Poor** | Rest or active recovery only |

### 3.2 Daily Strain Target

- Use readiness, historical load, and upcoming events to set a target strain band.
- Convert strain band into sport‑specific prescriptions:
  time × HR zone / RPE × (optional) pace or power.

**Example (Prime day, runner, 5K focus):**

> Target strain: 14–16 (hard interval day). Run 10–15 min easy warm‑up, then
> 6 × 3 min at 5K pace with 2 min easy jog, cool down to total 45–55 min.

---

## 4. Coaching Logic

See [Coaching Logic](coaching-logic.md) for full rules.

### 4.1 User Configuration (Onboarding)

- Primary sports: running, cycling, strength, swimming, team sports, etc.
- Weekly availability: days and approximate time budget per day.
- Goal type per sport:
  - Maintain fitness
  - Performance (e.g., 5K, 10K, marathon, FTP boost, strength gain)
  - Body composition
  - Return from layoff/injury (low load ramp)

### 4.2 Weekly Template Generator

Generate a default weekly pattern by sport and goal, then modulate by readiness each day.

**Example (runner, 5 days/week, performance goal):**

| Day | Session Type |
|-----|-------------|
| Mon | Easy aerobic |
| Tue | High‑intensity VO2/threshold |
| Wed | Easy / technique |
| Thu | Tempo or cross‑training |
| Fri | Rest |
| Sat | Long endurance |
| Sun | Optional strength |

On a "Prime" readiness day → promote hardest planned session.
On a "Low" readiness day → push hard session later, replace with easy/rest.

---

## 5. Surfaces: UX Flows

See [UX Flows](ux-flows.md) for wireframes and screen details.

### 5.1 Home / Today Screen

- Readiness score + color + one‑sentence insight
- Today's recommended session (time, intensity, focus)
- "Too tired?" / "Feeling fresh?" adjustment buttons

### 5.2 Workout Detail Screen

- Warm‑up, main set, cool‑down with target zones
- "Why this today" explanation linked to readiness + goal
- Export/sync to Garmin (future)

### 5.3 History / Trends (Minimal)

- 7‑ and 28‑day readiness vs strain charts
- Sleep and HRV trends with contextual annotations

---

## 6. AI/Chat Layer (Optional for MVP)

Stripped‑down "coach chat" focused on existing metrics.

| Intent | Response |
|--------|----------|
| "What should I do today?" | Same as Today panel + explanation |
| "Why is my readiness low?" | Highlight factors: bad sleep, high stress, heavy session |
| "Adjust for race in 10 days" | Mini‑taper: reduced volume, maintained intensity |

**Guardrails:** strictly answer based on stored metrics and rules;
no speculative medical advice.

---

## 7. Architecture

See [Architecture](architecture.md) for full details.

- **Backend:** Garmin ingestion microservice + coaching engine service
- **Storage:** Relational DB (Postgres) + time‑series tables
- **Frontend:** React Native / Expo (mobile) + Next.js (web)
- **Data privacy:** Explicit consent, transparent score computation

---

## 8. What "Done" Looks Like

A single Garmin user can:

1. ✅ Connect their Garmin account → see 30 days of backfilled data within minutes
2. ✅ See a daily readiness score and color with one‑line explanation
3. ✅ Get one main workout recommendation per day that:
   - Matches their chosen sport and goal
   - Adjusts volume and intensity with readiness
4. ✅ Ask basic questions ("why low?", "harder/easier?") → coherent, data‑backed answers
