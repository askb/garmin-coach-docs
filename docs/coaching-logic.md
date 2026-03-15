# Coaching Logic — GarminCoach

Deterministic rules + simple personalization. No complex ML for MVP.

---

## 1. Onboarding Configuration

### 1.1 Required Fields

```typescript
interface UserOnboarding {
  primarySports: SportType[];      // At least one
  weeklyAvailability: {
    days: DayOfWeek[];             // e.g., ['mon','tue','thu','sat']
    minutesPerDay: number;         // Average available training time
  };
  goals: {
    sport: SportType;
    goalType: GoalType;
    target?: string;               // e.g., "sub-20 5K", "bench 100kg"
  }[];
}

type SportType = 'running' | 'cycling' | 'strength' | 'swimming' | 'team_sport' | 'other';
type GoalType = 'maintain' | 'performance' | 'body_composition' | 'return_from_injury';
```

### 1.2 Derived Settings

From onboarding, compute:
- **HR zones** (Karvonen method: target = resting + % × (max − resting))
- **Pace zones** from recent activities or VO2max estimate
- **Weekly volume target** based on experience level and goal

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

### 2.2 Example Templates

#### `running-performance-5d` (Runner, Performance, 5 days)

| Day Slot (0‑indexed Mon–Sun) | Session Type | Intensity | Duration |
|-------------------------------|-------------|-----------|----------|
| 0 (Mon) | easy_run | easy | 30–45 min |
| 1 (Tue) | vo2_intervals | very_hard | 40–55 min |
| 2 (Wed) | recovery_jog | easy | 20–30 min |
| 3 (Thu) | tempo_run | hard | 35–50 min |
| 5 (Sat) | long_run | moderate | 60–90 min |

#### `strength-performance-3d` (Strength, Performance, 3 days)

| Day Slot (0‑indexed Mon–Sun) | Session Type | Intensity | Duration |
|-------------------------------|-------------|-----------|----------|
| 0 (Mon) | upper_heavy | hard | 45–60 min |
| 2 (Wed) | lower_heavy | hard | 45–60 min |
| 4 (Fri) | full_body | moderate | 40–50 min |

#### `cycling-performance-5d` (Cycling, Performance, 5 days)

| Day Slot (0‑indexed Mon–Sun) | Session Type | Intensity | Duration |
|-------------------------------|-------------|-----------|----------|
| 0 (Mon) | endurance_ride | easy | 60–90 min |
| 1 (Tue) | sweet_spot | hard | 50–70 min |
| 2 (Wed) | recovery_spin | easy | 30–40 min |
| 3 (Thu) | vo2_intervals | very_hard | 45–60 min |
| 5 (Sat) | endurance_ride | easy | 60–90 min |

### 2.3 Day Assignment

Map template slots to user's available days:

```
available_days = user.weeklyAvailability.days  // e.g., [mon, tue, thu, sat, sun]
template_slots = template.slots                // ordered by priority

// Assign hardest sessions to days with most time available
// Assign long session to weekend day if available
// Ensure at least 1 easy day between hard days
```

---

## 3. Daily Readiness Modulation

Each morning, adjust the planned workout based on readiness:

### 3.1 Modulation Rules

```typescript
function modulateWorkout(
  template: WorkoutTemplate,
  readinessZone: ReadinessZone,
  sport: string,
): WorkoutRecommendation {

  switch (readinessZone) {
    case 'prime':
      // If easy day planned, look for harder session later in week to swap
      // Otherwise intensify: +5% duration
      return applyDurationScale(template, 1.05);

    case 'high':
      // Execute as planned (1.0× duration)
      return applyDurationScale(template, 1.0);

    case 'moderate':
      // Reduce duration ×0.9
      return applyDurationScale(template, 0.9);

    case 'low':
      // Substitute hard/very_hard sessions with easy template
      if (template.intensity === 'hard' || template.intensity === 'very_hard') {
        return getEasyTemplate(sport);
      }
      return applyDurationScale(template, 1.0);

    case 'poor':
      // Return rest recommendation (0–20 min Zone 1)
      return getRestRecommendation();
  }
}
```

### 3.2 Hard Day Stacking Prevention

Hard day stacking is checked **before** modulation in `generateDailyWorkout`:

```
if (recentHardDays >= 2 && readinessZone !== 'prime') {
  // Force easy day after 2+ consecutive hard days
  return easyTemplate;
}
```

---

## 4. Workout Template Library

### 4.1 Running Templates

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `run-easy` | Easy Run | easy | 2 | 30–45 min | 4–7 |
| `run-recovery` | Recovery Jog | easy | 1–2 | 20–30 min | 2–4 |
| `run-tempo` | Tempo Run | hard | 3–4 | 35–50 min | 10–14 |
| `run-intervals-vo2` | VO2max Intervals | very_hard | 4–5 | 40–55 min | 14–18 |
| `run-long` | Long Run | moderate | 2 | 60–90 min | 10–14 |
| `run-strides` | Easy + Strides | easy | 2–5 | 35–45 min | 5–8 |
| `run-fartlek` | Fartlek Run | moderate | 2–4 | 35–50 min | 8–12 |

### 4.2 Cycling Templates

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `cycle-endurance` | Endurance Ride | easy | 2 | 60–90 min | 6–10 |
| `cycle-recovery` | Recovery Spin | easy | 1 | 30–40 min | 2–4 |
| `cycle-sweetspot` | Sweet Spot Intervals | hard | 3–4 | 50–70 min | 10–14 |
| `cycle-vo2` | VO2max Intervals | very_hard | 5 | 45–60 min | 14–18 |

### 4.3 Strength Templates

| ID | Name | Intensity | HR Zone | Duration | Strain |
|----|------|-----------|---------|----------|--------|
| `str-upper-heavy` | Upper Heavy | hard | 2–3 | 45–60 min | 8–12 |
| `str-lower-heavy` | Lower Heavy | hard | 2–4 | 45–60 min | 10–14 |
| `str-full-moderate` | Full Body | moderate | 2–3 | 40–50 min | 7–10 |
| `str-deload` | Deload Session | easy | 1–2 | 30–40 min | 3–5 |

### 4.4 Difficulty Adjustment

```typescript
export function adjustDifficulty(
  current: WorkoutRecommendation,
  direction: "harder" | "easier",
  sport: string,
): WorkoutRecommendation {
  if (direction === "easier") {
    // Reduce duration by 20%, drop HR zones by 1
    ...
  }
  // Harder: increase duration 10%, push HR zones up by 1
  ...
}
```

---

## 5. Parameterization from History

### 5.1 Pace/Power Targets

```
recent_best_5k_pace = getBestPace('5K', last_6_weeks);
estimated_paces = {
  easy:      recent_best_5k_pace × 1.25,  // ~25% slower
  tempo:     recent_best_5k_pace × 1.08,  // ~8% slower
  threshold: recent_best_5k_pace × 1.03,  // ~3% slower
  interval:  recent_best_5k_pace × 0.97,  // ~3% faster
  sprint:    recent_best_5k_pace × 0.90,  // ~10% faster
}

// Or use Garmin VO2max to estimate via Daniels' tables
if (vo2max) {
  paces = danielsTable.getPaces(vo2max);
}
```

### 5.2 Volume Progression

```
last_week_volume = getTotalVolume(last_7_days);
max_this_week = last_week_volume × 1.10;  // Cap at +10%

// Exception: if volume < 60% of 4‑week average, allow +15%
if (last_week_volume < fourWeekAvg × 0.6) {
  max_this_week = last_week_volume × 1.15;
}

// Deload every 4th week: target 60–70% of peak week
if (weekNumber % 4 === 0) {
  max_this_week = peakWeekVolume × 0.65;
}
```

### 5.3 Strength Progression

```
// Auto‑progress when all prescribed reps completed
if (completedAllSets(exercise, last_session)) {
  next_weight = last_weight + increment;  // 2.5kg upper, 5kg lower
}

// Auto‑reduce on low readiness
if (readinessZone === 'low') {
  working_weight = normal_weight × 0.85;
  sets = max(sets - 1, 2);
}
```

---

## 6. Team / Skill Sport Handling

```
// If user logs a "match" or "scrimmage" activity
if (activity.subType in ['match', 'scrimmage', 'game']) {
  // Classify as high strain (14+)
  // Auto‑schedule easy/rest days before and after
  adjustSurroundingDays(activity.date, { before: 'easy', after: 'rest_or_easy' });
}
```

---

## 7. Coach Chat Intent Handling

### Supported Intents

| Intent | Trigger Phrases | Handler |
|--------|----------------|---------|
| `daily_recommendation` | "What should I do today?" | Return today's workout + explanation |
| `readiness_explanation` | "Why is my readiness low/high?" | Return top factors from readiness computation |
| `adjust_difficulty` | "Make it harder/easier" | Re‑generate with ±1 zone shift |
| `race_taper` | "I have a race in N days" | Generate mini‑taper plan |
| `weekly_summary` | "How was my week?" | Aggregate strain, readiness, compliance |

### Guardrails

- Only reference data from the user's own metrics
- Never provide medical advice (injury, illness, medication)
- Acknowledge uncertainty: "Based on your data, I'd suggest…"
- Offer to escalate: "Consider consulting a coach or doctor if…"
