# Sport Science Reference — GarminCoach Training Engine

> **Purpose:** Every algorithm in the GarminCoach engine MUST cite peer-reviewed research.
> This document is the single source of truth for formulas, constants, thresholds, and
> their supporting evidence. Code reviews that introduce new training logic without a
> corresponding entry here will be rejected.

---

## Table of Contents

1. [Training Load (TRIMP / Strain)](#1-training-load-trimp--strain)
2. [Acute:Chronic Workload Ratio (ACWR)](#2-acutechronic-workload-ratio-acwr)
3. [Chronic Training Load (CTL / ATL / TSB)](#3-chronic-training-load-ctl--atl--tsb--banister-fitness-fatigue-model)
4. [HRV and Readiness](#4-hrv-and-readiness)
5. [Resting Heart Rate](#5-resting-heart-rate)
6. [Sleep Science](#6-sleep-science)
7. [VO2max Estimation](#7-vo2max-estimation)
8. [Race Prediction](#8-race-prediction)
9. [Training Status Classification](#9-training-status-classification)
10. [Recovery Time Estimation](#10-recovery-time-estimation)
11. [Periodization Models](#11-periodization-models)
12. [Running Economy & Biomechanics](#12-running-economy--biomechanics)
13. [Validation Methodology](#13-validation-methodology)
14. [Limitations & Disclaimers](#14-limitations--disclaimers)

---

## 1. Training Load (TRIMP / Strain)

Training impulse (TRIMP) quantifies the internal training load of a session by
combining duration and intensity into a single number. The engine supports four
TRIMP variants; the appropriate model is selected based on available sensor data.

### 1.1 Banister TRIMP (impulse-response)

**Formula:**

```
TRIMP = D × ΔHR_ratio × e^(k × ΔHR_ratio)

where:
  D           = session duration (minutes)
  ΔHR_ratio   = (HR_avg − HR_rest) / (HR_max − HR_rest)
  k           = 1.92  (male)
  k           = 1.67  (female)
  e           = Euler's number (≈ 2.71828)
```

The sex-specific weighting factor *k* accounts for the observed difference in
blood-lactate response curves between males and females at equivalent relative
heart-rate intensities.

**Engine constant:** `BANISTER_K_MALE = 1.92`, `BANISTER_K_FEMALE = 1.67`

> **Citation:** Banister EW. Modeling elite athletic performance. In: Green HJ,
> McDougal JD, Wenger HA, eds. *Physiological Testing of Elite Athletes*.
> Champaign, IL: Human Kinetics; 1991:403–424.

### 1.2 Edwards TRIMP (zone-weighted minutes)

Accumulates time-in-zone multiplied by a zone-specific weighting factor.

| HR Zone | % HRmax Range | Weight |
|---------|---------------|--------|
| Zone 1  | 50–60 %       | 1      |
| Zone 2  | 60–70 %       | 2      |
| Zone 3  | 70–80 %       | 3      |
| Zone 4  | 80–90 %       | 4      |
| Zone 5  | 90–100 %      | 5      |

**Formula:**

```
Edwards_TRIMP = Σ (minutes_in_zone_i × weight_i)   for i = 1..5
```

**Engine constant:** `EDWARDS_ZONE_WEIGHTS = [1, 2, 3, 4, 5]`

> **Citation:** Edwards S. High performance training and racing. In: *The Heart
> Rate Monitor Book*. Sacramento, CA: Feet Fleet Press; 1993:113–123.

### 1.3 Lucia TRIMP (3-zone ventilatory-threshold model)

A simplified model anchored to individually determined ventilatory thresholds
(VT1 and VT2), reducing the five-zone model to three physiologically distinct
intensity domains.

| Zone                 | Boundary     | Weight |
|----------------------|--------------|--------|
| Zone I  (below VT1)  | < VT1        | 1      |
| Zone II (VT1–VT2)    | VT1 – VT2   | 2      |
| Zone III (above VT2) | > VT2        | 3      |

**Formula:**

```
Lucia_TRIMP = Σ (minutes_in_zone_j × weight_j)   for j = I..III
```

> **Citation:** Lucia A, Hoyos J, Santalla A, Earnest C, Chicharro JL. Tour de
> France versus Vuelta a España: which is harder? *Med Sci Sports Exerc*.
> 2003;35(5):872–878.
>
> Original three-zone intensity classification described in: Lucia A, Hoyos J,
> Pérez M, Chicharro JL. Heart rate and performance parameters in elite
> cyclists: a longitudinal study. *Med Sci Sports Exerc*. 2000;32(10):1777–1782.
>
> Foundational efficiency data: Lucia A, Hoyos J, Pardo J, Chicharro JL.
> Metabolic and neuromuscular adaptations to endurance training in professional
> cyclists: a longitudinal study. *Jpn J Physiol*. 2000;50(3):381–388.

### 1.4 Session RPE (sRPE)

When heart-rate data is unavailable, session rating of perceived exertion (sRPE)
provides a practical alternative. Collected 30 minutes post-session using the
CR-10 scale.

**Formula:**

```
sRPE = RPE × duration (minutes)

where:
  RPE = Borg CR-10 rating (0–10 scale)
```

| RPE | Descriptor        |
|-----|-------------------|
| 0   | Rest              |
| 1   | Very easy         |
| 2   | Easy              |
| 3   | Moderate          |
| 4   | Somewhat hard     |
| 5   | Hard              |
| 6   | —                 |
| 7   | Very hard         |
| 8   | —                 |
| 9   | —                 |
| 10  | Maximal           |

> **Citation:** Foster C, Florhaug JA, Franklin J, et al. A new approach to
> monitoring exercise training. *J Strength Cond Res*. 2001;15(1):109–115.

---

## 2. Acute:Chronic Workload Ratio (ACWR)

The ACWR compares recent (acute, ~7 days) workload to longer-term habitual
(chronic, ~28 days) workload. It is the primary guard-rail in the engine for
injury-risk management.

### 2.1 Rolling-Average Model

```
ACWR = acute_load / chronic_load

where:
  acute_load   = sum of daily load over the last 7 days  / 7
  chronic_load = sum of daily load over the last 28 days / 28
```

### 2.2 Risk Zones

| ACWR Range | Classification      | Engine Action                         |
|------------|---------------------|---------------------------------------|
| < 0.8      | Under-prepared      | Flag under-training; suggest load ↑   |
| 0.8 – 1.3  | Sweet spot          | Continue as planned                   |
| 1.3 – 1.5  | Caution zone        | Cap intensity; add recovery day       |
| > 1.5      | Danger zone         | Hard cap; force recovery block        |

**Engine constants:**

```
ACWR_SWEET_SPOT_LOW  = 0.8
ACWR_SWEET_SPOT_HIGH = 1.3
ACWR_DANGER          = 1.5
```

> **Citations:**
>
> Blanch P, Gabbett TJ. Has the athlete trained enough to return to play
> safely? The acute:chronic workload ratio permits clinicians to quantify a
> player's risk of subsequent injury. *Br J Sports Med*. 2016;50(8):471–475.
>
> Hulin BT, Gabbett TJ, Blanch P, Chapman P, Bailey D, Orchard JW. Spikes in
> acute workload are associated with increased injury risk in elite cricket fast
> bowlers. *Br J Sports Med*. 2014;48(8):708–712.
>
> Hulin BT, Gabbett TJ, Lawson DW, Caputi P, Sampson JA. The acute:chronic
> workload ratio predicts injury: high chronic workload may decrease injury risk
> in elite rugby league players. *Br J Sports Med*. 2016;50(4):231–236.

### 2.3 Coupled vs Uncoupled ACWR

The **coupled** ACWR includes the acute period within the chronic calculation,
which introduces mathematical coupling — the acute period inflates the
denominator, biasing ACWR toward 1.0.

The **uncoupled** ACWR excludes the acute window from the chronic window
(days 8–28 only), removing this artefact.

```
uncoupled_chronic_load = sum of daily load days 8–28 / 21
uncoupled_ACWR         = acute_load / uncoupled_chronic_load
```

> **Citation:** Lolli L, Batterham AM, Hawkins R, et al. Mathematical coupling
> causes spurious correlation within the conventional acute-to-chronic workload
> ratio calculations. *Br J Sports Med*. 2019;53(1):54–55.

### 2.4 Exponentially Weighted Moving Average (EWMA) Model

EWMA gives more weight to recent sessions, providing a smoother and more
responsive estimate than the rolling average.

```
EWMA_today = Load_today × λ + EWMA_yesterday × (1 − λ)

where:
  λ_acute   = 2 / (N_acute  + 1) = 2 / (7  + 1) = 0.25
  λ_chronic = 2 / (N_chronic + 1) = 2 / (28 + 1) ≈ 0.069

ACWR_EWMA = EWMA_acute / EWMA_chronic
```

The EWMA approach reduces the artefact of old sessions "dropping off" the
rolling window and is the **preferred model in the engine**.

> **Citation:** Williams S, West S, Cross MJ, Stokes KA. Better way to
> determine the acute:chronic workload ratio? *Br J Sports Med*.
> 2017;51(3):209–210.

---

## 3. Chronic Training Load (CTL / ATL / TSB) — Banister Fitness-Fatigue Model

The performance model assumes that training produces two competing responses:
a slowly-decaying **fitness** effect and a rapidly-decaying **fatigue** effect.
The balance of these two effects predicts readiness to perform.

### 3.1 Definitions

| Metric | Full Name                  | Time Constant | Meaning               |
|--------|----------------------------|---------------|-----------------------|
| CTL    | Chronic Training Load      | τ₁ = 42 days  | Fitness               |
| ATL    | Acute Training Load        | τ₂ = 7 days   | Fatigue               |
| TSB    | Training Stress Balance    | CTL − ATL     | Form / freshness      |

### 3.2 Formulas

Exponential moving average (EMA) form:

```
CTL_today = CTL_yesterday + (TSS_today − CTL_yesterday) / τ₁
ATL_today = ATL_yesterday + (TSS_today − ATL_yesterday) / τ₂
TSB_today = CTL_today − ATL_today

where:
  τ₁ = 42  (fitness time constant, days)
  τ₂ = 7   (fatigue time constant, days)
  TSS_today = daily training stress score (TRIMP or equivalent)
```

Equivalently, using the decay factor *α*:

```
α₁ = e^(−1/42) ≈ 0.9765
α₂ = e^(−1/7)  ≈ 0.8669

CTL_today = α₁ × CTL_yesterday + (1 − α₁) × TSS_today
ATL_today = α₂ × ATL_yesterday + (1 − α₂) × TSS_today
```

**Engine constants:**

```
CTL_TAU = 42
ATL_TAU = 7
```

### 3.3 Interpreting TSB

| TSB Range    | Interpretation            |
|--------------|---------------------------|
| > +25        | Very fresh / possibly detraining |
| +10 to +25  | Fresh — good for racing    |
| −10 to +10  | Neutral — normal training  |
| −10 to −30  | Fatigued — building load   |
| < −30        | Very fatigued — overreaching risk |

> **Citations:**
>
> Banister EW, Calvert TW, Savage MV, Bach T. A systems model of training for
> athletic performance. *Aust J Sci Med Sport*. 1975;7:57–61.
>
> Busso T. Variable dose-response relationship between exercise training and
> performance. *Med Sci Sports Exerc*. 2003;35(7):1188–1195.
>
> Busso T, Denis C, Bonnefoy R, Geyssant A, Lacour JR. Modeling of adaptations
> to physical training by using a recursive least squares algorithm. *J Appl
> Physiol*. 1997;82(5):1685–1693.

---

## 4. HRV and Readiness

Heart-rate variability (HRV) reflects autonomic nervous system balance and is
the engine's primary physiological readiness marker. The natural logarithm of
the root-mean-square of successive differences (lnRMSSD) is the preferred metric
for athlete monitoring due to its superior reliability over SDNN in
non-stationary conditions typical of field recordings.

### 4.1 Preferred Metric: lnRMSSD

```
RMSSD = √( (1/N) × Σ (RR_i+1 − RR_i)² )

lnRMSSD = ln(RMSSD)
```

lnRMSSD is preferred over SDNN for athletes because:

- RMSSD predominantly reflects parasympathetic (vagal) modulation.
- The natural-log transformation normalises the distribution, enabling
  parametric statistical analysis.
- SDNN captures total variability (sympathetic + parasympathetic) and is more
  sensitive to recording length, making it less reliable for short morning
  recordings.

> **Citation:** Plews DJ, Laursen PB, Stanley J, Kilding AE, Buchheit M.
> Training adaptation and heart rate variability in elite endurance athletes:
> opening the door to effective monitoring. *Sports Med*. 2013;43(9):773–781.

### 4.2 Individual Z-Score Approach

Each athlete's daily lnRMSSD is compared against their own rolling baseline
(typically 7-day or 14-day window) to generate an individual z-score.

```
z = (lnRMSSD_today − mean_baseline) / SD_baseline

where:
  mean_baseline = rolling mean of lnRMSSD over the baseline window
  SD_baseline   = rolling SD of lnRMSSD over the baseline window
```

| z-Score     | Interpretation                      |
|-------------|-------------------------------------|
| > +1.0      | Parasympathetic saturation (or excellent recovery) |
| −1.0 to +1.0 | Normal range                      |
| < −1.0      | Sympathetic shift — potential fatigue/stress |
| < −2.0      | Strong suppression — flag for review |

> **Citation:** Plews DJ, Laursen PB, Kilding AE, Buchheit M. Training
> adaptation and heart rate variability in elite endurance athletes: opening the
> door to effective monitoring. *Int J Sports Physiol Perform*.
> 2013;8(6):688–694.

### 4.3 Coefficient of Variation (CV)

The CV of daily lnRMSSD over a rolling window reflects the stability of
autonomic regulation. A decreasing CV indicates positive training adaptation;
an increasing CV may indicate maladaptation.

```
CV = (SD_baseline / mean_baseline) × 100
```

| CV (lnRMSSD) | Interpretation                  |
|---------------|---------------------------------|
| < 3%          | Well-adapted, stable ANS        |
| 3–6%          | Normal variability              |
| > 6%          | Increased perturbation — review |

> **Citation:** Plews DJ, Laursen PB, Kilding AE, Buchheit M. Heart-rate
> variability and training-intensity distribution in elite rowers. *Int J Sports
> Physiol Perform*. 2014;9(6):1026–1032.

### 4.4 Smallest Worthwhile Change (SWC)

The SWC provides a threshold below which day-to-day changes in lnRMSSD are
likely noise rather than signal.

```
SWC = 0.5 × SD_baseline
```

A daily change exceeding the SWC (positive or negative) is flagged as a
meaningful deviation.

> **Citation:** Buchheit M. Monitoring training status with HR measures: do all
> roads lead to Rome? *Int J Sports Physiol Perform*. 2014;9(5):883–895.

### 4.5 HRV as Readiness Marker

The engine combines lnRMSSD z-score, CV trend, and SWC flags into a composite
readiness signal used by the adaptive scheduling system.

> **Citation:** Plews DJ, Laursen PB, Stanley J, Kilding AE, Buchheit M.
> Training adaptation and heart rate variability in elite endurance athletes:
> opening the door to effective monitoring. *Int J Sports Physiol Perform*.
> 2013;8(6):688–694.
>
> See also foundational HRV reliability work: Plews DJ, Laursen PB, Kilding AE,
> Buchheit M. Evaluating training adaptation with heart-rate measures: a
> methodological comparison. *Int J Sports Physiol Perform*. 2013;8(6):688–694.

---

## 5. Resting Heart Rate

Resting heart rate (RHR) complements HRV as a readiness indicator. While less
sensitive than HRV to day-to-day autonomic shifts, sustained RHR elevation
(≥2 consecutive days) is a well-established marker of incomplete recovery,
illness onset, or overtraining.

### 5.1 Detection Thresholds

```
RHR_deviation = RHR_today − RHR_baseline

where:
  RHR_baseline = rolling 14-day median of morning RHR
```

| Deviation         | Engine Action                                    |
|-------------------|--------------------------------------------------|
| +3 to +5 bpm     | Informational alert; no schedule change           |
| +5 to +10 bpm    | Suggest reduced intensity or extra recovery day   |
| > +10 bpm        | Flag potential illness/overtraining; force rest    |

A sustained elevation of 5–10 bpm above individual baseline is considered
clinically meaningful in the context of athlete monitoring.

### 5.2 Overtraining Syndrome Context

RHR elevation is one component of the broader overtraining syndrome (OTS)
diagnostic framework. OTS diagnosis requires a comprehensive approach including
performance decrements, mood disturbance, and hormonal markers — RHR alone is
insufficient.

> **Citations:**
>
> Meeusen R, Duclos M, Foster C, et al. Prevention, diagnosis, and treatment of
> the overtraining syndrome: joint consensus statement of the European College
> of Sport Science and the American College of Sports Medicine. *Eur J Sport
> Sci*. 2013;13(1):1–24.
>
> Hautala AJ, Kiviniemi AM, Tulppo MP. Individual responses to aerobic
> exercise: the role of the autonomic nervous system. *Neurosci Biobehav Rev*.
> 2009;33(2):107–115.
>
> Harriss DJ, Atkinson G. Ethical standards in sport and exercise science
> research: 2016 update. *Int J Sports Med*. 2015;36(14):1121–1124.

---

## 6. Sleep Science

Sleep is the primary recovery modality. The engine uses sleep data from Garmin
wearables (duration, stages, efficiency) to modulate recovery estimates and
readiness scores.

### 6.1 Duration Recommendations

**General adult population (18–64 years):**

| Organisation             | Recommended | Acceptable Range |
|--------------------------|-------------|------------------|
| National Sleep Foundation | 7–9 hours  | 6–10 hours       |
| AASM / SRS               | ≥ 7 hours  | —                |

> **Citations:**
>
> Hirshkowitz M, Whiton K, Albert SM, et al. National Sleep Foundation's sleep
> duration recommendations: methodology and results summary. *Sleep Health*.
> 2015;1(1):40–43.
>
> Watson NF, Badr MS, Belenky G, et al. Recommended amount of sleep for a
> healthy adult: a joint consensus statement of the American Academy of Sleep
> Medicine and Sleep Research Society. *Sleep*. 2015;38(6):843–844.

### 6.2 Athletes Need More Sleep

Athletes consistently benefit from extended sleep (9–10 hours) with measurable
performance improvements in reaction time, sprint speed, shooting accuracy, and
mood. The engine uses an athlete-specific recommendation of **8–10 hours**.

> **Citations:**
>
> Mah CD, Mah KE, Kezirian EJ, Dement WC. The effects of sleep extension on
> the athletic performance of collegiate basketball players. *Sleep*.
> 2011;34(7):943–950.
>
> Bird SP. Sleep, recovery, and athletic performance: a brief review and
> recommendations. *Strength Cond J*. 2013;35(5):43–47.

### 6.3 Sleep Stages — Normal Distributions

| Stage               | % of Total Sleep | Engine Threshold |
|---------------------|------------------|------------------|
| N1 (light)          | 2–5%             | Informational    |
| N2 (light)          | 45–55%           | Informational    |
| N3 (deep / SWS)     | 15–25%           | Flag if < 12%    |
| REM                 | 20–25%           | Flag if < 15%    |

Deep sleep (N3 / slow-wave sleep) is critical for growth-hormone release and
musculoskeletal recovery. REM sleep supports cognitive recovery, motor-skill
consolidation, and emotional regulation.

### 6.4 Sleep Efficiency

```
Sleep Efficiency (%) = (Total Sleep Time / Time in Bed) × 100
```

| Efficiency | Classification |
|------------|----------------|
| ≥ 85%      | Good           |
| 75–84%     | Fair           |
| < 75%      | Poor           |

> **Citation:** Dattilo M, Antunes HKM, Medeiros A, et al. Sleep and muscle
> recovery: endocrinological and molecular basis for a new and promising
> hypothesis. *Med Hypotheses*. 2011;77(2):220–222.

### 6.5 Sleep Debt

Cumulative sleep restriction creates a "sleep debt" that impairs cognitive and
physical performance in a dose-dependent manner. Importantly, recovery from
sleep debt is non-linear — a single night of extended sleep does not fully
recover performance.

The engine tracks a rolling 7-day sleep debt:

```
sleep_debt = Σ max(0, target_sleep − actual_sleep_i)   for i = 1..7

where:
  target_sleep = user-configured target (default 8 hours for athletes)
```

| Cumulative Debt (7d) | Engine Action                             |
|----------------------|-------------------------------------------|
| < 3 hours            | Normal — no adjustment                    |
| 3–7 hours            | Reduce next high-intensity session by 1 zone |
| > 7 hours            | Flag sleep-deprived; suggest recovery day   |

> **Citation:** Simpson NS, Dinges DF. Sleep and inflammation. *Nutr Rev*.
> 2007;65(suppl 3):S244–S252.
>
> Simpson NS, Gibbs EL, Matheson GO. Optimizing sleep to maximize performance:
> implications and recommendations for elite athletes. *Scand J Med Sci Sports*.
> 2017;27(3):266–274.
>
> Sleep debt recovery kinetics: Simpson NS, Dinges DF, Goel N, Van Dongen HPA.
> Repeating patterns of sleep restriction and recovery: do we get used to it?
> *Sleep*. 2016;39(3):693–700.

---

## 7. VO2max Estimation

VO2max (maximal oxygen uptake, ml·kg⁻¹·min⁻¹) is the gold-standard measure of
cardiorespiratory fitness. The engine uses multiple estimation methods depending
on available data.

### 7.1 ACSM Running Equation

For steady-state treadmill running:

```
VO2 = 3.5 + (0.2 × S) + (0.9 × S × G)

where:
  VO2 = oxygen consumption (ml·kg⁻¹·min⁻¹)
  S   = speed (m·min⁻¹)
  G   = fractional grade (e.g., 0.05 for 5% grade)
  3.5 = resting VO2 (1 MET)
```

For estimating VO2max from a maximal or near-maximal run:

```
VO2max ≈ VO2_at_pace / fraction_of_HRmax

where:
  fraction_of_HRmax = HR_exercise / HR_max
```

> **Citation:** American College of Sports Medicine. *ACSM's Guidelines for
> Exercise Testing and Prescription*. 11th ed. Philadelphia, PA: Wolters
> Kluwer; 2021.

### 7.2 Daniels' VDOT

VDOT is Jack Daniels' oxygen-cost-adjusted VO2max equivalent derived from race
performance. It accounts for both VO2max and running economy, making it more
predictive of performance than laboratory VO2max alone.

VDOT is determined via lookup tables mapping race time to a VDOT value. Training
paces are then derived from the VDOT:

| Training Zone     | % VDOT Pace | Purpose                |
|-------------------|-------------|------------------------|
| Easy (E)          | 59–74%      | Aerobic base           |
| Marathon (M)      | 75–84%      | Specific endurance     |
| Threshold (T)     | 83–88%      | Lactate clearance      |
| Interval (I)      | 95–100%     | VO2max stimulus        |
| Repetition (R)    | 105–110%    | Speed & economy        |

> **Citation:** Daniels J. *Daniels' Running Formula*. 3rd ed. Champaign, IL:
> Human Kinetics; 2013.

### 7.3 Cooper Test

A field test using 12-minute maximal-effort run distance:

```
VO2max = (d − 504.9) / 44.73

where:
  d = distance covered in 12 minutes (metres)
```

| Distance (m) | Estimated VO2max (ml·kg⁻¹·min⁻¹) |
|--------------|-----------------------------------|
| 2400         | 42.3                              |
| 2800         | 51.3                              |
| 3200         | 60.2                              |
| 3600         | 69.2                              |

> **Citation:** Cooper KH. A means of assessing maximal oxygen intake:
> correlation between field and treadmill testing. *JAMA*.
> 1968;203(3):201–204.

### 7.4 Uth Estimate (HR-ratio method)

A resting estimate requiring only HRmax and HRrest:

```
VO2max = 15.3 × (HRmax / HRrest)
```

| HRmax | HRrest | Estimated VO2max |
|-------|--------|------------------|
| 190   | 50     | 58.1             |
| 190   | 60     | 48.5             |
| 180   | 55     | 50.1             |
| 200   | 45     | 68.0             |

This method is useful for initial onboarding when no performance data is
available, but should be replaced by VDOT or ACSM estimates as data accumulates.

> **Citation:** Uth N, Sørensen H, Overgaard K, Pedersen PK. Estimation of
> VO2max from the ratio between HRmax and HRrest — the heart rate ratio method.
> *Eur J Appl Physiol*. 2004;91(1):111–115.

---

## 8. Race Prediction

### 8.1 Riegel Formula

The most widely used endurance prediction model, based on the observation that
performance fatigue follows a power-law relationship with distance.

```
T2 = T1 × (D2 / D1) ^ 1.06

where:
  T1 = known race time (seconds or minutes)
  D1 = known race distance
  D2 = target race distance
  1.06 = fatigue exponent (Riegel's constant)
```

> **Citation:** Riegel PS. Athletic records and human endurance. *Am Sci*.
> 1981;69(3):285–290.

### 8.2 Cameron Model

An alternative that uses a variable fatigue factor, providing better accuracy
for ultra-distance events (>42.2 km) where the constant 1.06 exponent
underestimates fatigue.

```
T2 = T1 × (D2 / D1) × (a + b × D2^c) / (a + b × D1^c)

where:
  a = 13.49681
  b = −0.000030363
  c = 1.06
  (Cameron's empirically derived constants)
```

> **Citation:** Cameron J. A proposed model for predicting 10 km to marathon
> performance. 1999. *(Online resource; model derived from analysis of
> age-group world records.)*

### 8.3 VDOT Tables

The most accurate prediction method for distances from 800 m to the marathon,
because VDOT incorporates running economy in addition to VO2max.

> **Citation:** Daniels J. *Daniels' Running Formula*. 3rd ed. Champaign, IL:
> Human Kinetics; 2013.

### 8.4 Example Predictions

Given a **20:00 5K** (VDOT ≈ 49):

| Target Distance    | Riegel (1.06) | VDOT Prediction | Cameron  |
|--------------------|---------------|-----------------|----------|
| 10K                | 41:34         | 41:26           | 41:30    |
| Half Marathon (21.1K) | 1:31:48    | 1:31:30         | 1:31:40  |
| Marathon (42.2K)   | 3:11:08       | 3:10:49         | 3:11:30  |

Given a **25:00 5K** (VDOT ≈ 39):

| Target Distance    | Riegel (1.06) | VDOT Prediction | Cameron  |
|--------------------|---------------|-----------------|----------|
| 10K                | 51:58         | 52:03           | 51:55    |
| Half Marathon (21.1K) | 1:54:33    | 1:55:55         | 1:54:45  |
| Marathon (42.2K)   | 3:58:40       | 4:01:19         | 3:59:10  |

*Note: Predictions assume equivalent training and race-day conditions. Longer
distances amplify pacing, nutrition, and heat-management errors.*

---

## 9. Training Status Classification

The engine assigns one of six training statuses based on the combination of
VO2max trend, ACWR, HRV trend, and load metrics.

### 9.1 Status Definitions

| Status        | VO2max Trend                | ACWR      | HRV Trend   | Description                           |
|---------------|-----------------------------|-----------|-------------|---------------------------------------|
| **Productive** | Trending up (>0.5 in 4 wk) | 0.8–1.3   | Stable/↑    | Fitness improving with appropriate load |
| **Maintaining**| Stable (±0.5 in 4 wk)      | 0.8–1.3   | Stable      | Holding fitness; adequate stimulus     |
| **Detraining** | Declining (>1.0 in 4 wk)   | < 0.6     | —           | Insufficient training stimulus         |
| **Overreaching**| Declining despite load     | > 1.5     | ↓ / high CV | High load but maladaptive response     |
| **Peaking**    | Stable or slight ↓          | 0.6–0.9   | ↑           | Intentional taper before goal event    |
| **Recovery**   | Not declining               | < 0.8     | Improving   | Low load after high-load block         |

### 9.2 Overreaching Subtypes

| Type | Duration   | Recovery     | Key Indicators                        |
|------|------------|--------------|---------------------------------------|
| Functional Overreaching (FOR) | Days to ~2 weeks | Full, with supercompensation | Temporary performance ↓, mood ↓ |
| Non-Functional Overreaching (NFOR) | Weeks to months | Slow, incomplete initially | Persistent performance ↓, sleep disruption, HRV suppression |
| Overtraining Syndrome (OTS) | Months+ | Prolonged; may require clinical intervention | Multi-system: hormonal, immunological, psychological |

> **Citations:**
>
> Meeusen R, Duclos M, Foster C, et al. Prevention, diagnosis, and treatment of
> the overtraining syndrome: joint consensus statement of the European College
> of Sport Science and the American College of Sports Medicine. *Eur J Sport
> Sci*. 2013;13(1):1–24.
>
> Halson SL, Jeukendrup AE. Does overtraining exist? An analysis of
> overreaching and overtraining research. *Sports Med*. 2004;34(14):967–981.

### 9.3 Peaking / Taper Protocol

An exponential taper of 2–3 weeks with volume reduction of 40–60% while
maintaining intensity and frequency produces optimal supercompensation.

```
daily_volume_taper = pre_taper_volume × e^(−t / τ_taper)

where:
  t        = days into taper
  τ_taper  = taper time constant (typically 5–7 days)
```

> **Citation:** Mujika I, Padilla S. Scientific bases for precompetition
> tapering strategies. *Med Sci Sports Exerc*. 2003;35(7):1182–1187.
>
> Bosquet L, Montpetit J, Arvisais D, Mujika I. Effects of tapering on
> performance: a meta-analysis. *Med Sci Sports Exerc*. 2007;39(8):1358–1365.

---

## 10. Recovery Time Estimation

### 10.1 Base Recovery Windows

Recovery time is assigned by session type (primary physiological demand) and
then adjusted by individual modifiers.

| Session Type              | Intensity Domain | Base Recovery (hours) |
|---------------------------|------------------|-----------------------|
| Recovery / easy run       | Zone 1–2         | 12–24                 |
| Long run (aerobic)        | Zone 1–2         | 24–48                 |
| Tempo / threshold         | Zone 3           | 24–48                 |
| VO2max intervals          | Zone 4–5         | 48–72                 |
| Speed / repetition        | Zone 5           | 48–72                 |
| Race (5K–10K)             | Maximal          | 72–96                 |
| Race (half marathon)      | Maximal          | 96–168 (4–7 days)     |
| Race (marathon)           | Maximal          | 168–336 (1–2 weeks)   |

### 10.2 Recovery Modifiers

The base window is scaled by multiplying with individual modifier coefficients:

```
adjusted_recovery = base_hours × Π(modifier_i)
```

| Modifier             | Coefficient Range | Source / Rationale                     |
|----------------------|-------------------|----------------------------------------|
| Sleep quality (poor) | 1.1 – 1.3        | Dattilo et al. 2011; Simpson et al. 2016 |
| Readiness z-score < −1 | 1.2 – 1.5     | Plews et al. 2013                      |
| Age > 40             | 1.1 – 1.2        | Fell & Williams, 2008                  |
| Age > 55             | 1.2 – 1.4        | Fell & Williams, 2008                  |
| Training age < 1 yr  | 1.2 – 1.3        | Conservative for beginners             |
| Training age > 5 yr  | 0.85 – 0.95      | Well-adapted athletes recover faster   |
| RHR elevation > 5 bpm | 1.2 – 1.4       | Meeusen et al. 2013                    |

> **Citations:**
>
> Hausswirth C, Mujika I, eds. *Recovery for Performance in Sport*. Champaign,
> IL: Human Kinetics; 2013.
>
> Fell J, Williams AD. The effect of aging on skeletal-muscle recovery from
> exercise: possible implications for aging athletes. *J Aging Phys Act*.
> 2008;16(1):97–115.

---

## 11. Periodization Models

### 11.1 Linear (Classical) Periodization

Progressive increase in volume followed by progressive increase in intensity,
organised into macrocycles (months), mesocycles (weeks), and microcycles (days).

**Structure:**

```
General Preparation → Specific Preparation → Competition → Transition
       (high volume,       (↑ intensity,      (peak/taper)   (recovery)
        low intensity)      ↓ volume)
```

> **Citation:** Bompa TO, Haff GG. *Periodization: Theory and Methodology of
> Training*. 6th ed. Champaign, IL: Human Kinetics; 2019.

### 11.2 Block Periodization

Concentrates training into highly focused blocks (2–4 weeks each) targeting a
single dominant fitness quality, allowing greater specificity and adaptation.

| Block            | Duration   | Focus                          |
|------------------|------------|--------------------------------|
| Accumulation     | 2–4 weeks  | Aerobic base, volume           |
| Transmutation    | 2–3 weeks  | Threshold / race-specific work |
| Realization      | 1–2 weeks  | Taper, speed, race prep        |

> **Citation:** Issurin VB. New horizons for the methodology and physiology of
> training periodization. *Sports Med*. 2010;40(3):189–206.

### 11.3 Polarized Training Distribution

The distribution of training intensity follows an approximately 80/20 split
between low-intensity (below VT1) and high-intensity (above VT2) work, with
minimal time in the moderate "black hole" zone (VT1–VT2).

```
Polarized distribution:
  ~80% of sessions/volume at low intensity  (below VT1 / Zone 1)
  ~0–5% at moderate intensity               (VT1–VT2 / Zone 2)
  ~15–20% at high intensity                 (above VT2 / Zone 3+)
```

This model has been validated across multiple endurance sports (rowing, cycling,
cross-country skiing, running) and consistently outperforms threshold-dominant
training in controlled studies.

### 11.4 Polarization Index (Quantifying Distribution)

The polarization index (PI) quantifies how closely an athlete's training
distribution follows the polarized model. Based on Shannon entropy applied
to a 3-zone classification (easy / moderate / hard):

**Formula:**

```
PI = ln(1 / Σ pᵢ²)

where:
  pᵢ = fraction of total training time in zone i
  Zones: easy (Zone 1–2), moderate (Zone 3), hard (Zone 4–5)
```

**Interpretation:**

| PI Value | Distribution | Interpretation |
|----------|-------------|---------------|
| > 2.0 | Highly polarized | Strong 80/20 split — ideal for most endurance athletes |
| 1.5–2.0 | Moderately polarized | Good distribution with some moderate-zone drift |
| 1.0–1.5 | Threshold-dominant | Too much Zone 3 — "black hole" training |
| < 1.0 | Single-zone dominant | Monotonic training — limited adaptation |

The ideal polarized distribution targets approximately:
- **~80%** of sessions/volume at easy intensity (below VT1)
- **~0–5%** at moderate intensity (VT1–VT2)
- **~15–20%** at high intensity (above VT2)

**Engine implementation:** The `zones.getPolarizationIndex()` endpoint calculates
PI from `hr_zone_minutes` JSONB data on the Activity table, classifying Zone 1–2
as easy, Zone 3 as moderate, and Zone 4–5 as hard.

> **Citation:** Seiler KS, Kjerland GØ. Quantifying training intensity
> distribution in elite endurance athletes: is there evidence for an "optimal"
> distribution? *Scand J Med Sci Sports*. 2006;16(1):49–56.

### 11.5 Aerobic Efficiency Index (Cardiac Drift)

The aerobic efficiency index tracks cardiac cost of locomotion over time.
An improving trend indicates enhanced running economy and/or aerobic adaptation.

**Formula:**

```
Efficiency Index = speed(m/s) / avgHr × 1000
```

**Interpretation:**

- Rising EI at constant HR → improved aerobic fitness
- Declining EI → possible overtraining, dehydration, or cardiac drift
- Useful for detecting aerobic adaptations that precede VO2max improvements

The efficiency index is related to the concept of cardiac drift — the gradual
increase in heart rate during prolonged exercise at constant workload — first
described by Coyle & González-Alonso (2001).

**Engine implementation:** The `zones.getEfficiencyTrend()` endpoint calculates
EI for each running/cycling activity and plots the trend as a scatter chart.

> **References:**
>
> Coyle EF, González-Alonso J. Cardiovascular drift during prolonged exercise:
> new perspectives. *Exerc Sport Sci Rev*. 2001;29(2):88–92.
>
> Seiler KS, Kjerland GØ. *Scand J Med Sci Sports*. 2006;16(1):49–56.

> **Citations:**
>
> Seiler KS, Kjerland GØ. Quantifying training intensity distribution in elite
> endurance athletes: is there evidence for an "optimal" distribution? *Scand J
> Med Sci Sports*. 2006;16(1):49–56.
>
> Stöggl T, Sperlich B. Polarized training has greater impact on key endurance
> variables than threshold, high intensity, or high volume training. *Front
> Physiol*. 2014;5:33.
>
> Neal CM, Hunter AM, Brennan L, et al. Six weeks of a polarized training-
> intensity distribution leads to greater physiological and performance
> adaptations than a threshold model in trained cyclists. *J Appl Physiol*.
> 2013;114(4):461–471.

---

## 12. Running Economy & Biomechanics

Running economy (RE) — the oxygen cost at a given submaximal speed — is a
critical determinant of distance-running performance, independent of VO2max.
The engine uses Garmin accelerometer-derived biomechanical metrics as proxies
for RE.

### 12.1 Ground Contact Time (GCT)

| Athlete Level | Typical GCT (ms) | Engine Reference |
|---------------|-------------------|------------------|
| Elite         | 160–200           | Optimal zone     |
| Sub-elite     | 200–240           | Good             |
| Recreational  | 250–300           | Room to improve  |

Shorter GCT generally correlates with better running economy, though the
relationship is mediated by leg stiffness and stride frequency.

> **Citation:** Santos-Concejero J, Granados C, Irazusta J, et al. Interaction
> effects of stride angle and strike pattern on running economy. *Int J Sports
> Med*. 2014;35(13):1118–1123.

### 12.2 Vertical Oscillation (VO)

| Athlete Level | Typical VO (cm) | Engine Reference |
|---------------|------------------|------------------|
| Elite         | 4–6              | Optimal zone     |
| Sub-elite     | 6–8              | Good             |
| Recreational  | 8–12             | Room to improve  |

Excessive vertical oscillation represents wasted energy in the vertical plane
that does not contribute to forward propulsion.

> **Citation:** Moore IS. Is there an economical running technique? A review of
> modifiable biomechanical factors affecting running economy. *Sports Med*.
> 2016;46(6):793–807.

### 12.3 Stride Length Optimisation

Self-selected stride length is typically within ±3% of the metabolically optimal
stride length. Conscious alterations (either lengthening or shortening) tend to
increase oxygen cost.

```
Optimal stride length ≈ self-selected stride length ± 3%
```

The engine flags stride length deviations > 5% from the user's recent baseline
as a potential economy concern (e.g., fatigue-induced overstriding).

> **Citation:** Cavanagh PR, Williams KR. The effect of stride length variation
> on oxygen uptake during distance running. *Med Sci Sports Exerc*.
> 1982;14(1):30–35.

---

## 13. Validation Methodology

All engine algorithms must pass the following validation gates before deployment.

### 13.1 Known-Answer Testing

Each TRIMP variant is validated against published training-load calculations
from the original papers. For Banister TRIMP, test vectors include the example
calculations in Banister (1991) Chapter 12, Table 12.1.

**Pass criterion:** Engine output matches published value within ±1%.

### 13.2 Cross-Validation Against Garmin Training Readiness

The composite readiness score is compared against Garmin's proprietary Training
Readiness metric using a paired dataset of ≥100 athlete-days.

**Pass criterion:** Pearson correlation *r* > 0.70 between engine readiness
score and Garmin Training Readiness.

### 13.3 Sensitivity Analysis

Each configurable weight and threshold is perturbed by ±10%. The resulting zone
or status classification is compared to the unperturbed output.

**Pass criterion:** A ±10% perturbation of any single weight must not change the
final zone/status classification by more than 1 zone or 1 status level.

### 13.4 Backtesting

Historical training data (≥6 months, ≥3 athletes) is replayed through the
engine. The generated recovery and load recommendations are compared against
actual outcomes.

**Pass criteria:**

- No "hard rest" recommendation on a day the athlete completed an easy session
  without issue.
- No "all clear" recommendation on a day within 48 hours preceding a reported
  injury or illness.
- Predicted recovery time correlates with actual next-session performance
  (r > 0.5).

### 13.5 Face Validity (Coach Review)

A panel of ≥3 certified coaches reviews a sample of 50 engine-generated training
recommendations and rates each as "agree," "partially agree," or "disagree."

**Pass criterion:** ≥80% "agree" or "partially agree" rating.

---

## 14. Limitations & Disclaimers

> **This application provides training guidance based on published sport science
> research. It is NOT a substitute for medical advice, clinical exercise testing,
> or professional coaching.**

### Key Limitations

1. **Population-level evidence, individual responses:** All formulas, constants,
   and thresholds in this document are derived from group-mean data in published
   studies. Individual physiological responses to training vary significantly
   due to genetics, training history, nutrition, stress, and other factors.

2. **Evidence-informed heuristics:** The weights and thresholds used in the
   engine (e.g., ACWR zones, recovery multipliers, sleep thresholds) are
   evidence-informed heuristics — they are grounded in research but are not
   clinically validated diagnostic cutoffs.

3. **Sensor limitations:** Wrist-based optical heart-rate sensors have known
   accuracy limitations during high-intensity exercise, particularly at high
   wrist movement frequencies. HRV measurements from consumer wearables may
   differ from clinical-grade ECG recordings.

4. **VO2max estimates are estimates:** Without laboratory gas-exchange testing,
   all VO2max values are approximations. The Uth method has a standard error of
   estimate of ±5 ml·kg⁻¹·min⁻¹; VDOT-derived estimates are more accurate but
   still approximate.

5. **Medical conditions:** Users with cardiovascular disease, metabolic
   conditions, musculoskeletal disorders, or other medical conditions should
   consult their healthcare provider before following any training
   recommendations generated by this engine. The engine does not screen for
   contraindications to exercise.

6. **Algorithm evolution:** Sport science is a living field. Formulas and
   thresholds will be updated as new evidence emerges. All changes will be
   version-tracked in this document.

---

## Revision History

| Date       | Version | Author | Changes                                    |
|------------|---------|--------|--------------------------------------------|
| 2025-01-01 | 1.0.0   | —      | Initial release — full reference document  |
| 2025-07-05 | 1.1.0   | —      | Added polarization index formula (§11.4), aerobic efficiency index (§11.5) |

---

*End of document.*
