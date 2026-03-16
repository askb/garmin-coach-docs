# Engine Reference — GarminCoach v2.0

> Every algorithm in the engine is evidence-based with peer-reviewed citations.
> See [Sport Science Reference](sport-science-reference.md) for full academic context.

**131 tests** — All modules fully covered by Vitest unit tests.

---

## 1. Readiness Scoring (Buchheit 2014)

Z-score based composite readiness scoring using 6 physiological components.

### 1.1 Component Scores

Each input produces a sub-score (0–100), then weighted and combined.

#### Sleep Quantity (20% weight)

```
sleep_ratio = actual_sleep_minutes / sleep_baseline_minutes
sleep_quantity_score =
  if sleep_ratio >= 1.0 → 100
  if sleep_ratio >= 0.85 → 70 + (sleep_ratio − 0.85) / 0.15 × 30
  if sleep_ratio >= 0.70 → 40 + (sleep_ratio − 0.70) / 0.15 × 30
  else → sleep_ratio / 0.70 × 40
```

#### Sleep Quality (15% weight)

```
deep_rem_ratio = (deep_sleep + rem_sleep) / total_sleep
efficiency = (total_sleep − awake_time) / time_in_bed
sleep_quality_score =
  garmin_sleep_score if available
  else (deep_rem_ratio × 60 + efficiency × 40)  # normalized to 0–100
```

#### HRV vs Baseline (25% weight — highest weight)

```
z_hrv = (today_hrv − hrv_baseline_mean) / hrv_baseline_sd
hrv_score =
  if z_hrv >= 1.0  → 100
  if z_hrv >= 0.0  → 80 + z_hrv × 20
  if z_hrv >= −1.0 → 60 + (z_hrv + 1.0) × 20
  if z_hrv >= −2.0 → 30 + (z_hrv + 2.0) × 30
  else → max(0, 30 + z_hrv × 15)
```

> Citation: Buchheit M. Monitoring training status with HR measures: do all roads
> lead to Rome? *Int J Sports Physiol Perform*. 2014;9(5):883–895.

#### Resting HR vs Baseline (10% weight)

```
rhr_delta = today_rhr − rhr_baseline
rhr_score =
  if rhr_delta <= −3 → 100   # Well below baseline
  if rhr_delta <= 0  → 80 + abs(rhr_delta) / 3 × 20
  if rhr_delta <= 3  → 60 − (rhr_delta / 3) × 20
  if rhr_delta <= 7  → 30 − ((rhr_delta − 3) / 4) × 30
  else → max(0, 10 − (rhr_delta − 7) × 2)
```

#### Training Load / ACWR (20% weight)

```
acwr = acute_load_7d / chronic_load_28d
load_score =
  if acwr >= 0.8 AND acwr <= 1.3 → 80 + (1.0 − abs(acwr − 1.05)) × 40
  if acwr < 0.8 → 70     # Under-trained, still okay
  if acwr > 1.3 AND acwr <= 1.5 → 50 − (acwr − 1.3) × 100
  if acwr > 1.5 → max(0, 30 − (acwr − 1.5) × 60)

# Consecutive hard day penalty
if consecutive_hard_days >= 3: load_score −= 15
if consecutive_hard_days >= 2: load_score −= 5
```

#### Stress & Body Battery (10% weight)

```
stress_normalized = 100 − garmin_stress_score
body_battery_score = body_battery_morning_value
stress_bb_score = stress_normalized × 0.4 + body_battery_score × 0.6
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

| Score | Zone | Color | Guidance |
|-------|------|-------|----------|
| 80–100 | **Prime** | Green | High-intensity or long workouts |
| 60–79 | **Good** | Blue-green | Normal training, optional intensity |
| 40–59 | **Moderate** | Yellow | Standard or slightly reduced |
| 20–39 | **Low** | Orange | Easy/technique only |
| 0–19 | **Poor** | Red | Rest or active recovery |

### 1.4 Confidence Levels

| Level | Criteria |
|-------|----------|
| **High** | All 6 components have data, baseline ≥ 14 days |
| **Medium** | 4–5 components available, baseline 7–13 days |
| **Low** | < 4 components or baseline < 7 days |

### 1.5 Explanation Generator

Identifies top 2 contributing factors (positive or negative deviation):

```
"HRV 12% above baseline and 8h quality sleep → Prime day for intensity."
"High training load (3 hard days) and stress elevated → take it easy today."
```

---

## 2. Strain & TRIMP (Banister 1991)

### 2.1 Banister TRIMP

```
TRIMP = D × ΔHR_ratio × e^(k × ΔHR_ratio)
where:
  D           = session duration (minutes)
  ΔHR_ratio   = (HR_avg − HR_rest) / (HR_max − HR_rest)
  k           = 1.92 (male), 1.67 (female)
```

> Citation: Banister EW. Modeling elite athletic performance. In: *Physiological
> Testing of Elite Athletes*. Human Kinetics; 1991:403–424.

### 2.2 Strain Score (0–21 Scale)

```
strain = 21 × (1 − e^(−trimp / personal_trimp_max))
```

Mapped to qualitative levels:

| Strain | Level | Typical Session |
|--------|-------|----------------|
| 0–4 | Light | Recovery jog, easy spin |
| 4–8 | Moderate | Easy run, endurance ride |
| 8–14 | Hard | Tempo, sweet spot, strength |
| 14–18 | Very Hard | VO2max intervals, race simulation |
| 18–21 | Maximal | Race effort, max test |

---

## 3. ACWR — Acute:Chronic Workload Ratio (Hulin 2016)

### 3.1 Rolling Average Model

```
ACWR = (Σ daily_load[last 7 days] / 7) / (Σ daily_load[last 28 days] / 28)
```

### 3.2 EWMA Model (Williams 2017, preferred)

```
EWMA_today = Load_today × λ + EWMA_yesterday × (1 − λ)
λ_acute   = 2 / (7 + 1)  = 0.25
λ_chronic = 2 / (28 + 1) ≈ 0.069
ACWR_EWMA = EWMA_acute / EWMA_chronic
```

### 3.3 Risk Zones

| ACWR | Zone | Engine Action |
|------|------|---------------|
| < 0.8 | Under-prepared | Flag under-training; suggest load increase |
| 0.8–1.3 | Sweet spot | Continue as planned |
| 1.3–1.5 | Caution | Cap intensity; add recovery day |
| > 1.5 | Danger | Hard cap on load; force recovery block |

> Citations: Hulin BT et al. *Br J Sports Med*. 2016;50(4):231–236.
> Williams S et al. *Br J Sports Med*. 2017;51(3):209–210.

---

## 4. CTL/ATL/TSB — Fitness-Fatigue Model (Banister 1975)

### 4.1 Definitions

| Metric | Full Name | Time Constant | Meaning |
|--------|-----------|---------------|---------|
| CTL | Chronic Training Load | τ₁ = 42 days | Fitness |
| ATL | Acute Training Load | τ₂ = 7 days | Fatigue |
| TSB | Training Stress Balance | CTL − ATL | Form / freshness |

### 4.2 Formulas (EMA form)

```
CTL_today = CTL_yesterday + (TSS_today − CTL_yesterday) / 42
ATL_today = ATL_yesterday + (TSS_today − ATL_yesterday) / 7
TSB_today = CTL_today − ATL_today
```

### 4.3 TSB Interpretation

| TSB Range | Interpretation |
|-----------|---------------|
| > +25 | Very fresh / possibly detraining |
| +10 to +25 | Fresh — good for racing |
| −10 to +10 | Neutral — normal training |
| −10 to −30 | Fatigued — building load |
| < −30 | Very fatigued — overreaching risk |

### 4.4 Ramp Rate

```
ramp_rate = (CTL_today − CTL_7_days_ago) / 7
```

| Ramp Rate | Interpretation |
|-----------|---------------|
| < 3 TSS/day | Conservative ramp |
| 3–7 TSS/day | Normal progression |
| > 7 TSS/day | Aggressive — overreaching risk |

> Citations: Banister EW et al. *Aust J Sci Med Sport*. 1975;7:57–61.
> Busso T. *Med Sci Sports Exerc*. 2003;35(7):1188–1195.

---

## 5. Load Focus Classification

Classifies each session and rolling load distribution:

| Category | Criteria | Description |
|----------|----------|-------------|
| **Aerobic** | > 70% time in Zone 1–2 | Base/endurance focused |
| **Anaerobic** | > 30% time in Zone 4–5 | High-intensity focused |
| **Mixed** | Neither threshold met | Balanced distribution |

Used in the Training Load dashboard to show load composition over time.

---

## 6. Baselines (14-day Rolling EMA + SD)

### 6.1 EMA Calculation

```
new_baseline = α × today_value + (1 − α) × old_baseline
α = 2 / (14 + 1) ≈ 0.133
```

### 6.2 Standard Deviation Tracking

Rolling SD calculated alongside EMA for z-score infrastructure:

```
z_score = (today_value − baseline_mean) / baseline_sd
```

### 6.3 Cold Start — Population Defaults (Shaffer & Ginsberg 2017)

| Metric | Age 18–29 | Age 30–39 | Age 40–49 | Age 50+ |
|--------|-----------|-----------|-----------|---------|
| HRV baseline (ms) | 55 | 45 | 38 | 30 |
| RHR baseline (bpm) | 62 | 64 | 66 | 68 |
| Sleep baseline (min) | 450 | 435 | 420 | 405 |

### 6.4 Baseline Blending

```
effective_baseline = personal_weight × personal + (1 − personal_weight) × population
personal_weight = min(1.0, days_of_data / 14)
```

> Citation: Shaffer F, Ginsberg JP. An overview of heart rate variability
> metrics and norms. *Front Public Health*. 2017;5:258.

---

## 7. Anomaly Detection

Z-score based detection with severity levels:

| Anomaly | Detection | Severity | Action |
|---------|-----------|----------|--------|
| **HRV drop** | z < −1.5 for 2+ days | Warning (2d) / Critical (3d+) | Suggest deload — reduce volume 40–50% |
| **RHR spike** | > 5 bpm above baseline for 2+ days | Warning (2d) / Critical (3d+) | Reduce planned intensity |
| **Sleep disruption** | < 6h for 3+ consecutive nights | Critical | Force rest day |
| **Overreaching** | ACWR > 1.5 + HRV declining | Critical | Cap strain at 8, suggest recovery block |

---

## 8. VO2max Estimation (3 Methods)

### 8.1 ACSM Running Equation

```
VO2 = 3.5 + (0.2 × S) + (0.9 × S × G)
VO2max ≈ VO2_at_pace / (HR_exercise / HR_max)
where: S = speed (m/min), G = grade fraction
```

> Citation: ACSM. *Guidelines for Exercise Testing and Prescription*. 11th ed. 2021.

### 8.2 Uth Ratio Method

```
VO2max = 15.3 × (HRmax / HRrest)
```

> Citation: Uth N et al. *Eur J Appl Physiol*. 2004;91(1):111–115.

### 8.3 Cooper Test

```
VO2max = (distance_12min_meters − 504.9) / 44.73
```

> Citation: Cooper KH. *JAMA*. 1968;203(3):201–204.

---

## 9. Race Prediction

### 9.1 Riegel Formula

```
T₂ = T₁ × (D₂ / D₁)^1.06
```

> Citation: Riegel PS. *Am Sci*. 1981;69(3):285–290.

### 9.2 VDOT Tables (Daniels)

VDOT incorporates running economy in addition to VO2max. Maps race times to
equivalent performances across distances.

> Citation: Daniels J. *Daniels' Running Formula*. 3rd ed. 2013.

### 9.3 Supported Distances

5K, 10K, half marathon (21.1K), marathon (42.2K)

### 9.4 Example Output

Given a 20:00 5K (VDOT ≈ 49):

| Distance | Riegel | VDOT |
|----------|--------|------|
| 10K | 41:34 | 41:26 |
| Half marathon | 1:31:48 | 1:31:30 |
| Marathon | 3:11:08 | 3:10:49 |

---

## 10. Training Status Classification

### 10.1 Six States

| Status | VO2max Trend | ACWR | HRV Trend | Description |
|--------|-------------|------|-----------|-------------|
| **Productive** | ↑ (>0.5 in 4 wk) | 0.8–1.3 | Stable/↑ | Fitness improving |
| **Maintaining** | Stable (±0.5) | 0.8–1.3 | Stable | Holding fitness |
| **Detraining** | ↓ (>1.0 in 4 wk) | < 0.6 | — | Insufficient stimulus |
| **Overreaching** | ↓ despite load | > 1.5 | ↓ / high CV | Maladaptive response |
| **Peaking** | Stable/slight ↓ | 0.6–0.9 | ↑ | Intentional taper |
| **Recovery** | Not declining | < 0.8 | Improving | Post-block recovery |

### 10.2 Overreaching Subtypes

| Type | Duration | Recovery |
|------|----------|----------|
| Functional (FOR) | Days to ~2 weeks | Full, with supercompensation |
| Non-functional (NFOR) | Weeks to months | Slow, incomplete |
| Overtraining syndrome (OTS) | Months+ | Prolonged, clinical |

> Citation: Meeusen R et al. *Eur J Sport Sci*. 2013;13(1):1–24.

---

## 11. Recovery Time Estimation (Hausswirth & Mujika 2013)

### 11.1 Base Recovery by Session Type

| Session Type | Intensity | Base Recovery (hours) |
|-------------|-----------|----------------------|
| Recovery / easy | Zone 1–2 | 12–24 |
| Long run | Zone 1–2 | 24–48 |
| Tempo / threshold | Zone 3 | 24–48 |
| VO2max intervals | Zone 4–5 | 48–72 |
| Race (5K–10K) | Maximal | 72–96 |
| Race (marathon) | Maximal | 168–336 |

### 11.2 Modifiers

```
adjusted_recovery = base_hours × Π(modifier_i)
```

| Modifier | Coefficient | Source |
|----------|------------|--------|
| Sleep quality (poor) | 1.1–1.3 | Dattilo et al. 2011 |
| Readiness z < −1 | 1.2–1.5 | Plews et al. 2013 |
| Age > 40 | 1.1–1.2 | Fell & Williams 2008 |
| Age > 55 | 1.2–1.4 | Fell & Williams 2008 |
| Training age < 1 yr | 1.2–1.3 | Conservative |
| Training age > 5 yr | 0.85–0.95 | Adapted athletes |
| Sleep debt > 3h | 1.1–1.2 | Simpson et al. 2017 |

> Citation: Hausswirth C, Mujika I, eds. *Recovery for Performance in Sport*.
> Human Kinetics; 2013.

---

## 12. Sleep Coach

### 12.1 Sleep Need Calculation

```
base_need = age_adjusted_need()
  18–25: 480 min (8h)
  26–64: 450 min (7.5h)
  65+:   420 min (7h)

athlete_adjustment = +60 min   # Athletes benefit from 8–10h (Mah 2011)

strain_adjustment =
  if yesterday_strain > 14: +30 min
  if yesterday_strain > 10: +15 min

total_need = base_need + athlete_adjustment + strain_adjustment
```

### 12.2 Sleep Debt Tracking

```
daily_debt = max(0, total_need − actual_sleep)
rolling_debt_7d = Σ daily_debt[last 7 days]
```

| Debt (7d) | Action |
|-----------|--------|
| < 180 min (3h) | Normal |
| 180–420 min (3–7h) | Reduce next high-intensity by 1 zone |
| > 420 min (7h) | Flag sleep-deprived; suggest recovery day |

### 12.3 Bedtime Recommendations

```
recommended_bedtime = target_wake_time − total_need − sleep_onset_latency
sleep_onset_latency = 15 min (default)
```

> Citation: Mah CD et al. *Sleep*. 2011;34(7):943–950.

---

## 13. Trend Analysis

### 13.1 Linear Regression

For any metric over a time window:

```
slope = Σ((x − x̄)(y − ȳ)) / Σ((x − x̄)²)
r² = correlation coefficient squared
p = significance (two-tailed t-test)
```

**Direction classification:**
- Slope > 0 and p < 0.05 → **Improving**
- Slope < 0 and p < 0.05 → **Declining**
- Otherwise → **Stable**

### 13.2 Rolling Averages

7-day and 28-day smoothed values for all tracked metrics.

### 13.3 Notable Change Detection

Flags when a metric's recent value deviates significantly from its trend:
- > 2 SD from 14-day mean → **Notable change** (positive or negative)
- Shown in the Trends UI as annotated markers

---

## 14. Correlations

Pearson r with p-values for 6 standard metric pairs:

| Pair | Expected Direction | Interpretation |
|------|-------------------|----------------|
| Sleep duration → Readiness | Positive | More sleep → higher readiness |
| HRV → Readiness | Positive | Higher HRV → higher readiness |
| Training load → Readiness | Negative | Higher load → lower readiness |
| Sleep duration → HRV | Positive | More sleep → better HRV |
| Stress → Readiness | Negative | Higher stress → lower readiness |
| RHR → Readiness | Negative | Higher RHR → lower readiness |

Calculated over configurable periods: 7d, 14d, 28d, 90d.

---

## 15. Running Form Analysis

### 15.1 Metrics and Benchmarks

| Metric | Elite | Sub-Elite | Recreational | Unit |
|--------|-------|-----------|-------------|------|
| Ground contact time (GCT) | 160–200 | 200–240 | 250–300 | ms |
| Vertical oscillation | 4–6 | 6–8 | 8–12 | cm |
| Stride length | Individual | Individual | Individual | m |
| Cadence | 170–185 | 165–175 | 155–170 | spm |
| GCT balance | 50 ± 1% | 50 ± 2% | 50 ± 3% | % |

### 15.2 Analysis Output

For each activity with running dynamics data:
- Comparison against elite benchmarks
- Comparison against personal recent averages
- Asymmetry detection (GCT balance deviation)
- Fatigue signature (GCT increase over activity duration)

> Citation: Moore IS. *Sports Med*. 2016;46(6):793–807.

---

## 16. Coaching Engine

### 16.1 Workout Templates (16 total)

**Running (7):**
`run-easy` · `run-recovery` · `run-tempo` · `run-intervals-vo2` · `run-long` · `run-strides` · `run-fartlek`

**Cycling (4):**
`cycle-endurance` · `cycle-recovery` · `cycle-sweetspot` · `cycle-vo2`

**Strength (5):**
`str-upper-heavy` · `str-lower-heavy` · `str-full-moderate` · `str-deload` · `str-hiit`

### 16.2 Weekly Planner

```
selectWeeklyTemplate(sport, goalType, availableDays) → WeekSlot[]
```

Maps sport × goal × days/week to an optimal weekly distribution:
- Hardest sessions separated by 48+ hours
- Long sessions on weekends
- At least 1 easy day between hard days
- Deload every 4th week (60–70% volume)

### 16.3 Readiness-Based Modulation

```
modulateWorkout(template, readinessZone, sport) → WorkoutRecommendation
```

| Zone | Modulation |
|------|-----------|
| Prime | Promote harder session or +5% duration |
| Good | Execute as planned |
| Moderate | −10% duration |
| Low | Substitute hard → easy template |
| Poor | Rest or 20 min Zone 1 only |

### 16.4 Hard Day Stacking Prevention

```
if recentHardDays >= 2 AND readinessZone !== 'prime':
  force easy day
```

### 16.5 Difficulty Adjustment

User can request harder/easier adjustments:
- **Easier:** −20% duration, drop HR zones by 1
- **Harder:** +10% duration, push HR zones up by 1
