# Readiness & Strain Engine — GarminCoach

The "WHOOP Brain" of the application. Uses transparent, deterministic scoring
rather than opaque ML for the MVP.

---

## 1. Daily Readiness Score (0–100)

### 1.1 Component Scores

Each input produces a sub‑score from 0–100, then weighted and combined.

#### Sleep Quantity (20% weight)

```
sleep_ratio = actual_sleep_minutes / sleep_baseline_minutes
sleep_quantity_score =
  if sleep_ratio >= 1.0 → 100
  if sleep_ratio >= 0.85 → 70 + (sleep_ratio - 0.85) / 0.15 × 30
  if sleep_ratio >= 0.70 → 40 + (sleep_ratio - 0.70) / 0.15 × 30
  else → sleep_ratio / 0.70 × 40
```

#### Sleep Quality (15% weight)

```
# Use Garmin sleep score if available, otherwise compute:
deep_rem_ratio = (deep_sleep + rem_sleep) / total_sleep
efficiency = (total_sleep - awake_time) / time_in_bed

sleep_quality_score =
  garmin_sleep_score if available
  else (deep_rem_ratio × 60 + efficiency × 40)   # normalized to 0–100
```

#### HRV vs Baseline (25% weight)

```
hrv_ratio = today_hrv / hrv_30day_baseline
hrv_score =
  if hrv_ratio >= 1.10 → 100
  if hrv_ratio >= 1.00 → 80 + (hrv_ratio - 1.00) / 0.10 × 20
  if hrv_ratio >= 0.90 → 60 + (hrv_ratio - 0.90) / 0.10 × 20
  if hrv_ratio >= 0.75 → 30 + (hrv_ratio - 0.75) / 0.15 × 30
  else → hrv_ratio / 0.75 × 30
```

#### Resting HR vs Baseline (10% weight)

```
# Lower resting HR is better; deviation above baseline is bad
rhr_delta = today_rhr - rhr_30day_baseline
rhr_score =
  if rhr_delta <= -3 → 100   # Well below baseline
  if rhr_delta <= 0  → 80 + abs(rhr_delta) / 3 × 20
  if rhr_delta <= 3  → 60 - (rhr_delta / 3) × 20
  if rhr_delta <= 7  → 30 - ((rhr_delta - 3) / 4) × 30
  else → max(0, 10 - (rhr_delta - 7) × 2)
```

#### Recent Training Load (20% weight)

```
# Compare 3‑day acute load to 7‑day chronic load
acute_load = sum(strain_scores[last_3_days]) / 3
chronic_load = sum(strain_scores[last_7_days]) / 7
acwr = acute_load / chronic_load   # Acute:Chronic Workload Ratio

load_score =
  if acwr >= 0.8 AND acwr <= 1.3 → 80 + (1.0 - abs(acwr - 1.05)) × 40
  if acwr < 0.8 → 70    # Under‑trained, still okay
  if acwr > 1.3 AND acwr <= 1.5 → 50 - (acwr - 1.3) × 100
  if acwr > 1.5 → max(0, 30 - (acwr - 1.5) × 60)

# Also penalize consecutive hard days
consecutive_hard_days = count_consecutive_days(strain > 14)
if consecutive_hard_days >= 3: load_score -= 15
if consecutive_hard_days >= 2: load_score -= 5
```

#### Stress & Body Battery (10% weight)

```
stress_normalized = 100 - garmin_stress_score  # Invert: low stress = high score
body_battery_score = body_battery_morning_value  # Already 0–100

stress_bb_score = (stress_normalized × 0.4 + body_battery_score × 0.6)
```

### 1.2 Final Readiness Calculation

```
readiness = round(
  sleep_quantity_score × 0.20 +
  sleep_quality_score  × 0.15 +
  hrv_score            × 0.25 +
  rhr_score            × 0.10 +
  load_score           × 0.20 +
  stress_bb_score      × 0.10
)

readiness = clamp(readiness, 0, 100)
```

### 1.3 Zone Mapping

```typescript
function getReadinessZone(score: number): ReadinessZone {
  if (score >= 80) return 'prime';
  if (score >= 60) return 'high';
  if (score >= 40) return 'moderate';
  if (score >= 20) return 'low';
  return 'poor';
}
```

### 1.4 Explanation Generator

Generate a one‑line explanation by identifying the top 2 contributing factors
(positive or negative deviation from baseline):

```
"HRV 12% above baseline and 8h quality sleep → Prime day for intensity."
"High training load (3 hard days) and stress elevated → take it easy today."
```

---

## 2. Daily Strain Target

### 2.1 Base Strain by Readiness Zone

| Zone | Target Strain Range (0–21) | Training Intent |
|------|---------------------------|-----------------|
| Prime | 14–18 | High intensity or long duration |
| High | 10–14 | Normal training, moderate intensity |
| Moderate | 7–11 | Standard or slightly reduced |
| Low | 4–7 | Easy/technique only |
| Poor | 0–4 | Rest or light movement |

### 2.2 Modifiers

```
base_strain = zone_strain_midpoint

# Goal modifier
if goal == 'performance': base_strain += 1
if goal == 'maintain': base_strain += 0
if goal == 'body_composition': base_strain += 0.5
if goal == 'return_from_injury': base_strain -= 2

# Load trend modifier (prevent overreaching)
if acwr > 1.3: strain_cap = min(base_strain, 10)
if consecutive_rest_days >= 2: base_strain += 1  # Safely push after rest

# Event proximity (if race entered)
if days_to_race <= 3: strain_cap = 6    # Taper
if days_to_race <= 7: base_strain -= 3  # Pre‑taper reduction
```

### 2.3 Strain‑to‑Workout Conversion

```
target_duration = strain_to_duration(target_strain, sport, intensity_zone)
target_hr_zone = strain_to_hr_zone(target_strain, readiness_zone)

# Example mappings:
# Strain 15, running → 45–55 min, Zone 4–5 intervals
# Strain 10, running → 40–50 min, Zone 2–3 easy
# Strain 6, cycling  → 30–40 min, Zone 1–2 recovery spin
```

---

## 3. Baseline Management

### 3.1 Rolling Baselines

All baselines use a 30‑day exponential moving average (EMA):

```
new_baseline = α × today_value + (1 - α) × old_baseline
α = 2 / (30 + 1) ≈ 0.0645
```

### 3.2 Cold Start (New Users)

For the first 7 days, use population defaults until enough data accumulates:

| Metric | Default (Male) | Default (Female) |
|--------|----------------|-------------------|
| HRV baseline | 45 ms | 50 ms |
| Resting HR baseline | 62 bpm | 65 bpm |
| Sleep baseline | 420 min (7h) | 420 min (7h) |
| Daily strain capacity | 12 | 11 |

After 7+ days, blend personal data with defaults:
```
effective_baseline = personal_weight × personal_baseline + (1 - personal_weight) × default
personal_weight = min(1.0, days_of_data / 30)
```

---

## 4. Anomaly Detection

Flag and surface alerts when:

| Anomaly | Threshold | Severity | Action |
|---------|-----------|----------|--------|
| HRV crash | > 25% below baseline for 2+ days | Warning (2d) / Critical (3d+) | Suggest deload week — reduce volume 40–50% |
| RHR spike | > 5 bpm above baseline for 2+ days | Warning (2d) / Critical (3d+) | Reduce planned intensity |
| Sleep deficit | < 6h for 3+ consecutive nights | Critical | Force rest day |
| Overreaching | ACWR > 1.5 | Critical | Cap strain at 8, suggest recovery |
