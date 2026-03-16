# Coaching Logic — GarminCoach v2.0

Deterministic, evidence-based coaching that adapts daily based on readiness,
training status, recovery state, and sleep quality.

---

## 1. Onboarding Configuration

### 1.1 Required Fields

```typescript
interface UserOnboarding {
  primarySports: SportType[];
  weeklyAvailability: {
    days: DayOfWeek[];
    minutesPerDay: number;
  };
  goals: {
    sport: SportType;
    goalType: GoalType;
    target?: string;           // e.g., "sub-20 5K", "bench 100kg"
  }[];
}

type SportType = 'running' | 'cycling' | 'strength' | 'swimming' | 'team_sport' | 'other';
type GoalType = 'maintain' | 'performance' | 'body_composition' | 'return_from_injury';
```

### 1.2 Derived Settings

From onboarding, the system computes:
- **HR zones** (Karvonen method: target = resting + % × (max − resting))
- **Pace zones** from recent activities or VO2max estimate (Daniels' VDOT tables)
- **Weekly volume target** based on experience level and goal
- **Training status baseline** for subsequent classification

---

## 2. Weekly Template Generator

### 2.1 Template Selection Matrix

| Sport | Goal | Days/Week | Template ID |
|-------|------|-----------|-------------|
| Running | Performance | 5 | `running-performance-5d` |
| Running | Performance | 3–4 | `running-performance-3d` |
| Running | Maintain | 3–4 | `running-maintain-3d` |
| Cycling | Performance | 4–5 | `cycling-performance-5d` |
| Cycling | Maintain | 3 | `cycling-maintain-3d` |
| Strength | Performance | 3–4 | `strength-performance-3d` |
| Strength | Body Comp | 4 | `strength-body_composition-4d` |

### 2.2 Day Assignment Rules

1. Assign hardest sessions to days with most time available
2. Long sessions on weekend days when possible
3. Ensure ≥ 1 easy day between hard days
4. 48+ hour separation between very hard sessions

### 2.3 Volume Progression

```
max_this_week = last_week_volume × 1.10   # Cap at +10%/week
if last_week_volume < four_week_avg × 0.6:
  max_this_week = last_week_volume × 1.15  # Allow +15% when rebuilding

# Deload every 4th week
if week_number % 4 === 0:
  max_this_week = peak_week_volume × 0.65
```

---

## 3. Daily Readiness Modulation

Each morning, adjust the planned workout based on readiness zone:

### 3.1 Modulation Rules

| Zone | Action | Duration Scale |
|------|--------|---------------|
| **Prime** | Promote harder session or intensify | 1.05× |
| **Good** | Execute as planned | 1.0× |
| **Moderate** | Reduce volume slightly | 0.9× |
| **Low** | Substitute hard → easy template | — |
| **Poor** | Return rest recommendation (0–20 min Zone 1) | — |

### 3.2 Hard Day Stacking Prevention

Checked **before** modulation in `generateDailyWorkout`:

```
if recentHardDays >= 2 AND readinessZone !== 'prime':
  return easyTemplate
```

### 3.3 Difficulty Adjustment (User Override)

Users can request harder/easier on any given day:
- **"Too tired"** → down-shift by 1 zone (e.g., tempo → easy run), −20% duration
- **"Feeling fresh"** → up-shift by 1 zone (e.g., easy → tempo with strides), +10% duration

---

## 4. Training Status Integration

Training status (from the engine's `classifyTrainingStatus()`) influences
coaching decisions:

| Status | Coaching Response |
|--------|------------------|
| **Productive** | Continue current plan; allow intensity progression |
| **Maintaining** | Consider adding a harder stimulus if goal is progression |
| **Detraining** | Increase frequency or intensity; alert user |
| **Overreaching** | Trigger forced recovery block (3–5 days easy/rest) |
| **Peaking** | Maintain intensity, reduce volume 40–60% (taper protocol) |
| **Recovery** | Low load, prioritize sleep and mobility work |

### 4.1 Taper Protocol (Pre-Race)

When user is in peaking status or approaching a goal event:

```
daily_volume = pre_taper_volume × e^(−t / τ_taper)
τ_taper = 5–7 days
```

- Volume: −40–60% from peak
- Intensity: Maintained (keep sharpness)
- Frequency: Slightly reduced (−1 day/week)

> Citation: Mujika I, Padilla S. *Med Sci Sports Exerc*. 2003;35(7):1182–1187.

---

## 5. Recovery Time Integration

The coaching system uses `estimateRecoveryTime()` from the engine to:

1. **Block hard sessions** if recovery time hasn't elapsed since last hard effort
2. **Adjust next workout** based on remaining recovery window
3. **Surface recovery estimates** in the Training Load dashboard

### Recovery Modifiers Applied

| Factor | Effect on Recovery |
|--------|-------------------|
| High strain session | +12–24h base |
| Poor readiness (z < −1) | +20–50% |
| Age > 40 | +10–20% |
| Sleep debt > 3h | +10–20% |
| Training age > 5 yr | −5–15% (adapted) |

---

## 6. Sleep Coach Integration

Sleep data feeds directly into coaching decisions:

### 6.1 Pre-Workout Checks

| Sleep Condition | Coaching Action |
|----------------|-----------------|
| Sleep debt > 7h (7d cumulative) | Force rest day |
| Sleep debt 3–7h | Downgrade intensity by 1 zone |
| Sleep efficiency < 75% (3+ nights) | Flag and suggest bedtime adjustment |
| Deep sleep < 12% (persistent) | Informational alert |

### 6.2 Bedtime Recommendations

```
recommended_bedtime = wake_time − total_sleep_need − onset_latency
total_sleep_need = base_need + athlete_adj + strain_adj
onset_latency = 15 min (default)
```

Surfaced in the Sleep Dashboard as actionable coaching advice.

---

## 7. Periodization Concepts

The coaching system implements simplified periodization:

### 7.1 Mesocycle Structure (4-Week Blocks)

| Week | Volume | Intensity | Purpose |
|------|--------|-----------|---------|
| 1 | Base | Moderate | Building |
| 2 | +5–10% | Moderate–Hard | Loading |
| 3 | +5–10% | Hard | Peak loading |
| 4 | −35–40% | Easy–Moderate | Deload / recovery |

### 7.2 Load Distribution (Polarized Model)

Following evidence from Seiler & Kjerland 2006:

```
~80% sessions at easy intensity (Zone 1–2)
~0–5% at moderate intensity (Zone 3)
~15–20% at high intensity (Zone 4–5)
```

The weekly planner automatically distributes sessions to approximate
this 80/20 split.

### 7.3 Race-Specific Phases

| Phase | Duration | Focus |
|-------|----------|-------|
| **Base** | 4–8 weeks | Aerobic foundation, volume building |
| **Build** | 4–6 weeks | Race-specific intensity, tempo, intervals |
| **Peak** | 1–2 weeks | Sharpening, race simulation |
| **Taper** | 1–2 weeks | Volume ↓ 40–60%, intensity maintained |

---

## 8. Workout Template Library (16 Templates)

### 8.1 Running Templates (7)

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `run-easy` | Easy Run | easy | 2 | 30–45 min | 4–7 |
| `run-recovery` | Recovery Jog | easy | 1–2 | 20–30 min | 2–4 |
| `run-tempo` | Tempo Run | hard | 3–4 | 35–50 min | 10–14 |
| `run-intervals-vo2` | VO2max Intervals | very_hard | 4–5 | 40–55 min | 14–18 |
| `run-long` | Long Run | moderate | 2 | 60–90 min | 10–14 |
| `run-strides` | Easy + Strides | easy | 2–5 | 35–45 min | 5–8 |
| `run-fartlek` | Fartlek Run | moderate | 2–4 | 35–50 min | 8–12 |

### 8.2 Cycling Templates (4)

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `cycle-endurance` | Endurance Ride | easy | 2 | 60–90 min | 6–10 |
| `cycle-recovery` | Recovery Spin | easy | 1 | 30–40 min | 2–4 |
| `cycle-sweetspot` | Sweet Spot | hard | 3–4 | 50–70 min | 10–14 |
| `cycle-vo2` | VO2max Intervals | very_hard | 5 | 45–60 min | 14–18 |

### 8.3 Strength Templates (5)

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `str-upper-heavy` | Upper Heavy | hard | 2–3 | 45–60 min | 8–12 |
| `str-lower-heavy` | Lower Heavy | hard | 2–4 | 45–60 min | 10–14 |
| `str-full-moderate` | Full Body | moderate | 2–3 | 40–50 min | 7–10 |
| `str-deload` | Deload Session | easy | 1–2 | 30–40 min | 3–5 |
| `str-hiit` | HIIT Circuit | very_hard | 4–5 | 25–35 min | 12–16 |

---

## 9. Parameterization from History

### 9.1 Pace/Power Targets

```
recent_best_5k_pace = getBestPace('5K', last_6_weeks)
estimated_paces = {
  easy:      recent_best_5k_pace × 1.25,
  tempo:     recent_best_5k_pace × 1.08,
  threshold: recent_best_5k_pace × 1.03,
  interval:  recent_best_5k_pace × 0.97,
  sprint:    recent_best_5k_pace × 0.90,
}

// Or use VO2max for Daniels' VDOT-based paces
if (vo2max) paces = danielsTable.getPaces(vo2max)
```

### 9.2 Strength Progression

```
if completedAllSets(exercise, last_session):
  next_weight = last_weight + increment  // 2.5kg upper, 5kg lower

if readinessZone === 'low':
  working_weight = normal_weight × 0.85
  sets = max(sets − 1, 2)
```

---

## 10. Team / Skill Sport Handling

```
if activity.subType in ['match', 'scrimmage', 'game']:
  // Classify as high strain (14+)
  // Auto-schedule recovery around match days
  adjustSurroundingDays(activity.date, { before: 'easy', after: 'rest_or_easy' })
```

---

## 11. Guardrails

- Only reference data from the user's own metrics
- Never provide medical advice (injury, illness, medication)
- Acknowledge uncertainty: "Based on your data, I'd suggest…"
- Offer escalation: "Consider consulting a coach or doctor if…"
- Volume caps: never exceed +15% week-over-week
- ACWR hard cap: force recovery when > 1.5
