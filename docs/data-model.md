# Data Model — GarminCoach v2.1

## Entity Relationship Overview

```
User 1──N Profile
User 1──N DailyMetric
User 1──N Activity
User 1──N ReadinessScore
User 1──N WeeklyPlan
User 1──N VO2maxEstimate
User 1──N TrainingStatus
User 1──N JournalEntry
User 1──N CorrelationResult
User 1──N RacePrediction
User 1──N ChatMessage
WeeklyPlan 1──N DailyWorkout
Activity 1──N WorkoutTimeSeries
```

> **Auth tables:** `user`, `session`, `account`, and `verification` are managed by
> [Better-Auth](https://www.better-auth.com/) and defined via its adapter. All
> other tables are defined with Drizzle ORM (`pgTable()`) in
> `packages/db/src/schema.ts`.

---

## Auth Tables (Better-Auth Managed)

### user

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| name | TEXT | Display name |
| email | TEXT | Unique |
| emailVerified | BOOLEAN | |
| image | TEXT | Avatar URL |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### session

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| expiresAt | TIMESTAMPTZ | |
| token | TEXT | Unique |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |
| ipAddress | TEXT | |
| userAgent | TEXT | |
| userId | TEXT | FK → user |

### account

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| accountId | TEXT | Provider-side ID |
| providerId | TEXT | e.g. "credential", "garmin", "discord" |
| userId | TEXT | FK → user |
| accessToken | TEXT | Encrypted at rest |
| refreshToken | TEXT | Encrypted at rest |
| idToken | TEXT | |
| accessTokenExpiresAt | TIMESTAMPTZ | |
| refreshTokenExpiresAt | TIMESTAMPTZ | |
| scope | TEXT | |
| password | TEXT | Hashed, for credential provider |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### verification

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| identifier | TEXT | |
| value | TEXT | |
| expiresAt | TIMESTAMPTZ | |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

---

## Application Tables (13 Tables)

### 1. Profile

Extends the auth user with athlete-specific configuration.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key (defaultRandom) |
| userId | TEXT | References user.id, unique |
| age | INT | |
| sex | VARCHAR(20) | 'male', 'female', 'other' |
| massKg | DOUBLE PRECISION | Body mass in kg |
| heightCm | DOUBLE PRECISION | Height in cm |
| timezone | VARCHAR(50) | IANA timezone (default 'UTC') |
| experienceLevel | VARCHAR(20) | 'beginner', 'intermediate', 'advanced' |
| primarySports | JSONB | String array: ['running', 'cycling', ...] |
| goals | JSONB | Array of {sport, goalType, target?} |
| weeklyDays | JSONB | String array of available days |
| minutesPerDay | INT | Default 45 |
| maxHr | INT | Measured or estimated (220−age) |
| restingHrBaseline | DOUBLE PRECISION | Rolling 14-day EMA |
| hrvBaseline | DOUBLE PRECISION | Rolling 14-day EMA |
| sleepBaseline | DOUBLE PRECISION | Rolling 14-day EMA |
| createdAt | TIMESTAMP | |
| updatedAt | TIMESTAMPTZ | Auto-updated |

### 2. DailyMetric (30+ fields)

Daily health data ingested from Garmin.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | References user.id |
| date | DATE | Unique per user |
| sleepScore | INT | 0–100 (Garmin or computed) |
| totalSleepMinutes | INT | Total sleep duration |
| deepSleepMinutes | INT | N3 / slow-wave sleep |
| remSleepMinutes | INT | REM sleep |
| lightSleepMinutes | INT | N1 + N2 sleep |
| awakeMinutes | INT | Time awake during sleep |
| sleepStartTime | TIMESTAMPTZ | Sleep onset time |
| sleepEndTime | TIMESTAMPTZ | Wake time |
| sleepEfficiency | DOUBLE PRECISION | % (TST / time in bed × 100) |
| hrv | DOUBLE PRECISION | Overnight average (ms) |
| hrvRmssd | DOUBLE PRECISION | Raw RMSSD value (ms) |
| restingHr | INT | Resting heart rate (bpm) |
| maxHr | INT | Max HR from day's activities |
| stressScore | INT | 0–100 (Garmin) |
| bodyBatteryStart | INT | Morning Body Battery value |
| bodyBatteryEnd | INT | Evening Body Battery value |
| steps | INT | Daily step count |
| calories | INT | Total calories |
| activeCalories | INT | Active calories |
| distance | DOUBLE PRECISION | Daily distance (meters) |
| floorsClimbed | INT | |
| intensity | INT | Garmin intensity minutes |
| garminTrainingReadiness | INT | NULL if device doesn't support |
| garminTrainingLoad | DOUBLE PRECISION | NULL if not available |
| garminSleepScore | INT | Garmin's computed sleep score |
| respirationRate | DOUBLE PRECISION | Breaths per minute |
| spo2 | DOUBLE PRECISION | Blood oxygen % |
| rawGarminData | JSONB | Full API response for debugging |
| syncedAt | TIMESTAMP | |

**Unique constraint:** `(userId, date)`

### 3. Activity (25+ fields)

Individual activity/workout records.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | References user.id |
| garminActivityId | VARCHAR(100) | Deduplication key |
| sportType | VARCHAR(50) | running, cycling, strength, swimming, etc. |
| subType | VARCHAR(50) | intervals, tempo, long_run, race, etc. |
| startedAt | TIMESTAMPTZ | Activity start |
| endedAt | TIMESTAMPTZ | Activity end |
| durationMinutes | DOUBLE PRECISION | Total duration |
| distanceMeters | DOUBLE PRECISION | NULL for strength |
| avgHr | INT | Average heart rate |
| maxHr | INT | Maximum heart rate |
| avgPaceSecPerKm | INT | NULL for non-distance sports |
| bestPaceSecPerKm | INT | Best split pace |
| avgCadence | INT | Steps/min or RPM |
| calories | INT | |
| elevationGain | DOUBLE PRECISION | Meters gained |
| trimpScore | DOUBLE PRECISION | Banister TRIMP |
| strainScore | DOUBLE PRECISION | 0–21 scale |
| vo2maxEstimate | DOUBLE PRECISION | From this activity |
| avgGct | DOUBLE PRECISION | Ground contact time (ms) |
| avgVerticalOscillation | DOUBLE PRECISION | Vertical oscillation (cm) |
| avgStrideLength | DOUBLE PRECISION | Stride length (m) |
| gctBalance | DOUBLE PRECISION | L/R balance (%) |
| trainingEffect | DOUBLE PRECISION | Garmin Training Effect |
| hrZoneMinutes | JSONB | HR zone time distribution (see below) |
| rawGarminData | JSONB | Full API response |
| syncedAt | TIMESTAMP | |

**Indexes:** `(userId, startedAt)`, `(garminActivityId)` UNIQUE

#### hr_zone_minutes JSONB Structure

Minutes spent in each of the 5 standard HR zones per activity. Used by the
Zone Analytics dashboard for distribution, polarization, and trend analysis.

```json
{
  "zone1": 12.5,    // Zone 1: 50–60% HRmax (recovery)
  "zone2": 18.3,    // Zone 2: 60–70% HRmax (aerobic base)
  "zone3": 8.1,     // Zone 3: 70–80% HRmax (tempo)
  "zone4": 4.2,     // Zone 4: 80–90% HRmax (threshold)
  "zone5": 1.9      // Zone 5: 90–100% HRmax (VO2max)
}
```

Values are in minutes (DOUBLE PRECISION). NULL when HR data is unavailable.

### 4. ReadinessScore

Computed daily readiness with full component breakdown.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| date | DATE | |
| score | INT | 0–100 |
| zone | VARCHAR(20) | prime, good, moderate, low, poor |
| confidence | VARCHAR(20) | high, medium, low (based on data completeness) |
| sleepQuantityComponent | DOUBLE PRECISION | Sub-score 0–100 |
| sleepQualityComponent | DOUBLE PRECISION | Sub-score 0–100 |
| hrvComponent | DOUBLE PRECISION | Sub-score 0–100 |
| restingHrComponent | DOUBLE PRECISION | Sub-score 0–100 |
| trainingLoadComponent | DOUBLE PRECISION | Sub-score 0–100 |
| stressComponent | DOUBLE PRECISION | Sub-score 0–100 |
| explanation | TEXT | Human-readable one-liner |
| factors | JSONB | Detailed breakdown |
| computedAt | TIMESTAMPTZ | |

**Unique constraint:** `(userId, date)`

### 5. WeeklyPlan

Weekly training plan template assignment.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| weekStart | DATE | Monday of the week |
| sport | VARCHAR(50) | |
| goalType | VARCHAR(50) | |
| template | JSONB | Base weekly template used |
| createdAt | TIMESTAMPTZ | |

### 6. DailyWorkout

Individual workout prescriptions within a weekly plan.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| weeklyPlanId | UUID | FK → weekly_plan |
| userId | TEXT | FK → user |
| date | DATE | |
| sportType | VARCHAR(50) | |
| workoutType | VARCHAR(50) | easy_run, intervals, tempo, long_run, etc. |
| title | VARCHAR(200) | "Easy Recovery Run" |
| description | TEXT | Full workout description |
| targetDurationMin | INT | Minimum duration (minutes) |
| targetDurationMax | INT | Maximum duration (minutes) |
| targetHrZoneLow | INT | 1–5 |
| targetHrZoneHigh | INT | 1–5 |
| targetStrainLow | DOUBLE PRECISION | |
| targetStrainHigh | DOUBLE PRECISION | |
| structure | JSONB | Warm-up, intervals, cool-down blocks |
| readinessZoneUsed | VARCHAR(20) | Zone when workout was generated |
| status | VARCHAR(20) | planned, completed, skipped, modified |
| explanation | TEXT | "Why this today" text |
| createdAt | TIMESTAMPTZ | |

**Index:** `(userId, date)`

### 7. ChatMessage

AI coach conversation history.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| role | VARCHAR(20) | 'user' or 'assistant' |
| content | TEXT | Message content |
| context | JSONB | Readiness, metrics used for response |
| createdAt | TIMESTAMPTZ | |

### 8. VO2maxEstimate

VO2max estimation history across multiple methods.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| date | DATE | |
| method | VARCHAR(50) | 'acsm', 'uth', 'cooper', 'garmin' |
| value | DOUBLE PRECISION | ml·kg⁻¹·min⁻¹ |
| confidence | VARCHAR(20) | high, medium, low |
| inputData | JSONB | Source data used for estimation |
| createdAt | TIMESTAMPTZ | |

### 9. TrainingStatus

Training status classification history.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| date | DATE | |
| status | VARCHAR(30) | productive, maintaining, detraining, overreaching, peaking, recovery |
| vo2maxTrend | DOUBLE PRECISION | Rate of change |
| acwr | DOUBLE PRECISION | Current ACWR value |
| hrvTrend | VARCHAR(20) | improving, stable, declining |
| ctl | DOUBLE PRECISION | Current CTL |
| atl | DOUBLE PRECISION | Current ATL |
| tsb | DOUBLE PRECISION | Current TSB |
| rampRate | DOUBLE PRECISION | CTL change rate |
| createdAt | TIMESTAMPTZ | |

### 10. JournalEntry

Athlete daily journal with mood and tagging.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| date | DATE | |
| content | TEXT | Free-form journal text |
| mood | VARCHAR(20) | great, good, neutral, tired, poor |
| tags | JSONB | String array: ['race', 'injury', 'travel', ...] |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### 11. CorrelationResult

Statistical correlations between metric pairs.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| metricPair | VARCHAR(100) | e.g. 'sleep→readiness', 'hrv→readiness' |
| rValue | DOUBLE PRECISION | Pearson correlation coefficient (−1 to +1) |
| pValue | DOUBLE PRECISION | Statistical significance |
| n | INT | Sample size |
| period | VARCHAR(20) | '7d', '14d', '28d', '90d' |
| computedAt | TIMESTAMPTZ | |

### 12. RacePrediction

Race time predictions from various methods.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | FK → user |
| inputDistance | VARCHAR(20) | e.g. '5K', '10K' |
| inputTime | INT | Seconds |
| targetDistance | VARCHAR(20) | e.g. '10K', 'half', 'marathon' |
| predictedTime | INT | Seconds |
| method | VARCHAR(20) | 'riegel', 'vdot', 'cameron' |
| vdot | DOUBLE PRECISION | VDOT value if applicable |
| createdAt | TIMESTAMPTZ | |

### 13. WorkoutTimeSeries

High-resolution time-series data for individual activities.

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| activityId | UUID | FK → activity |
| userId | TEXT | FK → user |
| timestamp | TIMESTAMPTZ | Data point time |
| heartRate | INT | BPM |
| pace | DOUBLE PRECISION | sec/km |
| cadence | INT | steps/min or RPM |
| power | DOUBLE PRECISION | Watts (if available) |
| altitude | DOUBLE PRECISION | Meters |
| gct | DOUBLE PRECISION | Ground contact time (ms) |
| verticalOscillation | DOUBLE PRECISION | cm |
| latitude | DOUBLE PRECISION | GPS latitude |
| longitude | DOUBLE PRECISION | GPS longitude |

**Index:** `(activityId, timestamp)`

---

## Computed Values

### TRIMP Score (Banister 1991)

```
TRIMP = duration_minutes × ΔHR_ratio × e^(k × ΔHR_ratio)
where:
  ΔHR_ratio = (avg_hr − resting_hr) / (max_hr − resting_hr)
  k = 1.92 (male) or 1.67 (female)
```

### Strain Score (0–21 scale)

```
strain = 21 × (1 − e^(−trimp / personal_trimp_max))
```

### ACWR (Hulin 2016)

```
ACWR = acute_load_7d / chronic_load_28d
EWMA variant (Williams 2017): λ_acute = 0.25, λ_chronic = 0.069
```

### CTL/ATL/TSB (Banister 1975)

```
CTL_today = CTL_yesterday + (TSS_today − CTL_yesterday) / 42
ATL_today = ATL_yesterday + (TSS_today − ATL_yesterday) / 7
TSB_today = CTL_today − ATL_today
```

### Rolling Baselines

- **Window:** 14-day exponential moving average (EMA)
- **EMA formula:** new = α × today + (1 − α) × old, where α = 2 / (14 + 1) ≈ 0.133
- **SD tracking:** Rolling standard deviation for z-score calculations
- **Cold start:** Age-adjusted population defaults (Shaffer & Ginsberg 2017)

---

## Garmin API Field Mapping

| Garmin API Field | Our Column |
|-----------------|------------|
| `summaryId` | daily_metric.garmin_id |
| `sleepDurationInSeconds` | daily_metric.totalSleepMinutes (÷60) |
| `deepSleepDurationInSeconds` | daily_metric.deepSleepMinutes (÷60) |
| `remSleepInSeconds` | daily_metric.remSleepMinutes (÷60) |
| `lightSleepDurationInSeconds` | daily_metric.lightSleepMinutes (÷60) |
| `awakeDurationInSeconds` | daily_metric.awakeMinutes (÷60) |
| `restingHeartRateInBeatsPerMinute` | daily_metric.restingHr |
| `averageStressLevel` | daily_metric.stressScore |
| `bodyBatteryChargedValue` | daily_metric.bodyBatteryStart |
| `bodyBatteryDrainedValue` | daily_metric.bodyBatteryEnd |
| `stepsGoal` / `steps` | daily_metric.steps |
| `activityType` | activity.sportType |
| `averageHeartRateInBeatsPerMinute` | activity.avgHr |
| `maxHeartRateInBeatsPerMinute` | activity.maxHr |
| `durationInSeconds` | activity.durationMinutes (÷60) |
| `distanceInMeters` | activity.distanceMeters |
| `vO2MaxValue` | activity.vo2maxEstimate |
| `averageRunCadence` | activity.avgCadence |
| `groundContactTime` | activity.avgGct |
| `verticalOscillation` | activity.avgVerticalOscillation |
| `strideLength` | activity.avgStrideLength |
