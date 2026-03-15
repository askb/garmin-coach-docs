# Data Model — GarminCoach

## Entity Relationship Overview

```
User 1──N Profile
User 1──N DailyMetric
User 1──N Activity
User 1──N ReadinessScore
User 1──N WeeklyPlan
WeeklyPlan 1──N DailyWorkout
```

> **Auth tables:** `user`, `session`, `account`, and `verification` are managed by
> [Better-Auth](https://www.better-auth.com/) and defined via its adapter. All
> other tables are defined with Drizzle ORM (`pgTable()`) in
> `packages/db/src/schema.ts`.

---

## Tables

### user *(Better-Auth managed)*

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| name | TEXT | Display name |
| email | TEXT | Unique |
| emailVerified | BOOLEAN | |
| image | TEXT | Avatar URL |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### session *(Better-Auth managed)*

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

### account *(Better-Auth managed)*

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| accountId | TEXT | Provider‑side ID |
| providerId | TEXT | e.g. "credential", "garmin" |
| userId | TEXT | FK → user |
| accessToken | TEXT | |
| refreshToken | TEXT | |
| idToken | TEXT | |
| accessTokenExpiresAt | TIMESTAMPTZ | |
| refreshTokenExpiresAt | TIMESTAMPTZ | |
| scope | TEXT | |
| password | TEXT | Hashed, for credential provider |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### verification *(Better-Auth managed)*

| Column | Type | Notes |
|--------|------|-------|
| id | TEXT | Primary key |
| identifier | TEXT | |
| value | TEXT | |
| expiresAt | TIMESTAMPTZ | |
| createdAt | TIMESTAMPTZ | |
| updatedAt | TIMESTAMPTZ | |

### profile

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key (defaultRandom) |
| userId | TEXT | References user.id |
| age | INT | |
| sex | ENUM('male','female','other') | |
| mass_kg | DECIMAL(5,2) | |
| height_cm | DECIMAL(5,2) | |
| timezone | VARCHAR(50) | IANA timezone |
| experience_level | ENUM('beginner','intermediate','advanced') | |
| primary_sports | JSONB | Array of sport enums |
| goals | JSONB | Array of {sport, goal_type, target?} |
| weeklyDays | JSONB | String array of available days |
| minutesPerDay | INT | Default 45 |
| max_hr | INT | Measured or estimated (220‑age) |
| resting_hr_baseline | DECIMAL(5,2) | Rolling 30‑day |
| hrv_baseline | DECIMAL(5,2) | Rolling 30‑day |
| sleepBaseline | DECIMAL(5,2) | Rolling sleep baseline |

### daily_metric

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | References user.id |
| date | DATE | Unique per user |
| sleep_score | INT | 0–100 (Garmin or computed) |
| total_sleep_minutes | INT | |
| deep_sleep_minutes | INT | |
| rem_sleep_minutes | INT | |
| light_sleep_minutes | INT | |
| awake_minutes | INT | |
| hrv | DECIMAL(5,2) | Overnight average ms |
| resting_hr | INT | bpm |
| max_hr | INT | bpm (from day's activities) |
| stress_score | INT | 0–100 (Garmin) |
| body_battery_start | INT | Morning value |
| body_battery_end | INT | Evening value |
| steps | INT | |
| calories | INT | |
| garmin_training_readiness | INT | NULL if device doesn't support |
| garmin_training_load | DECIMAL(6,2) | NULL if not available |
| raw_garmin_data | JSONB | Full API response for debugging |
| synced_at | TIMESTAMPTZ | |

**Unique constraint:** `(userId, date)`

### activity

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| userId | TEXT | References user.id |
| garmin_activity_id | VARCHAR(100) | Dedup key |
| sport_type | VARCHAR(50) | running, cycling, strength, swimming, etc. |
| sub_type | VARCHAR(50) | intervals, tempo, long_run, etc. |
| started_at | TIMESTAMPTZ | |
| ended_at | TIMESTAMPTZ | |
| duration_minutes | DECIMAL(6,2) | |
| distance_meters | DECIMAL(10,2) | NULL for strength |
| avg_hr | INT | |
| max_hr | INT | |
| avg_pace_sec_per_km | INT | NULL for non‑distance sports |
| calories | INT | |
| trimp_score | DECIMAL(6,2) | Computed: duration × HR intensity |
| strain_score | DECIMAL(4,2) | 0–21 WHOOP‑style scale |
| vo2max_estimate | DECIMAL(4,1) | |
| raw_garmin_data | JSONB | |
| synced_at | TIMESTAMPTZ | |

**Index:** `(user_id, started_at)`, `(garmin_activity_id)` UNIQUE

### readiness_score

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| user_id | UUID | FK → users |
| date | DATE | |
| score | INT | 0–100 |
| zone | ENUM('prime','high','moderate','low','poor') | |
| sleep_quantity_component | DECIMAL(4,2) | Sub‑score 0–100 |
| sleep_quality_component | DECIMAL(4,2) | |
| hrv_component | DECIMAL(4,2) | |
| resting_hr_component | DECIMAL(4,2) | |
| training_load_component | DECIMAL(4,2) | |
| stress_component | DECIMAL(4,2) | |
| explanation | TEXT | Human‑readable one‑liner |
| factors | JSONB | Detailed breakdown for chat |
| computed_at | TIMESTAMPTZ | |

**Index:** `(user_id, date)` UNIQUE

### weekly_plan

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| user_id | UUID | FK → users |
| week_start | DATE | Monday of the week |
| sport | VARCHAR(50) | |
| goal_type | VARCHAR(50) | |
| template | JSONB | Base weekly template used |
| created_at | TIMESTAMPTZ | |

### daily_workout

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| weekly_plan_id | UUID | FK → weekly_plans |
| user_id | UUID | FK → users |
| date | DATE | |
| sport_type | VARCHAR(50) | |
| workout_type | VARCHAR(50) | easy_run, intervals, tempo, long_run, etc. |
| title | VARCHAR(200) | "Easy Recovery Run" |
| description | TEXT | Full workout description |
| target_duration_min | INT | |
| target_duration_max | INT | |
| target_hr_zone_low | INT | 1–5 |
| target_hr_zone_high | INT | 1–5 |
| target_strain_low | DECIMAL(4,2) | |
| target_strain_high | DECIMAL(4,2) | |
| structure | JSONB | Warm‑up, intervals, cool‑down blocks |
| readiness_zone_used | VARCHAR(20) | Zone when workout was generated |
| status | ENUM('planned','completed','skipped','modified') | |
| explanation | TEXT | "Why this today" text |
| created_at | TIMESTAMPTZ | |

**Index:** `(user_id, date)`

### chat_message

| Column | Type | Notes |
|--------|------|-------|
| id | UUID | Primary key |
| user_id | UUID | FK → users |
| role | ENUM('user','assistant') | |
| content | TEXT | |
| context | JSONB | Readiness, metrics used for response |
| created_at | TIMESTAMPTZ | |

---

## Computed Values

### TRIMP Score

```
TRIMP = duration_minutes × ΔHR_ratio × e^(k × ΔHR_ratio)
where:
  ΔHR_ratio = (avg_hr - resting_hr) / (max_hr - resting_hr)
  k = 1.92 (male) or 1.67 (female)
```

### Strain Score (0–21 scale)

```
strain = 21 × (1 - e^(-trimp / personal_trimp_max))
```

### Rolling Baselines

- **HRV baseline:** 30‑day exponential moving average of overnight HRV
- **Resting HR baseline:** 30‑day EMA of resting HR
- **Sleep baseline:** 30‑day average total sleep minutes

---

## Garmin API Field Mapping

| Garmin API Field | Our Column |
|-----------------|------------|
| `summaryId` | daily_metric.garmin_id |
| `sleepDurationInSeconds` | daily_metric.total_sleep_minutes (÷60) |
| `deepSleepDurationInSeconds` | daily_metric.deep_sleep_minutes |
| `remSleepInSeconds` | daily_metric.rem_sleep_minutes |
| `restingHeartRateInBeatsPerMinute` | daily_metric.resting_hr |
| `averageStressLevel` | daily_metric.stress_score |
| `bodyBatteryChargedValue` | daily_metric.body_battery_start |
| `stepsGoal` / `steps` | daily_metric.steps |
| `activityType` | activity.sport_type |
| `averageHeartRateInBeatsPerMinute` | activity.avg_hr |
| `durationInSeconds` | activity.duration_minutes (÷60) |
| `distanceInMeters` | activity.distance_meters |
| `vO2MaxValue` | activity.vo2max_estimate |
