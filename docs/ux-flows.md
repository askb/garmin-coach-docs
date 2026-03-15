# UX Flows & Screens — GarminCoach

## Screen Map

```
/              → Home / Dashboard (readiness + workout)
/onboarding    → 3-step setup flow
/trends        → History / Trends (7d / 28d charts)
/settings      → Profile, Garmin, data privacy
/workout/[id]  → Workout detail (structure, targets, explanation)
```

---

## 1. Onboarding Flow

Progress bar shows 3 segments. No Garmin OAuth in onboarding. No confirmation screen.

### Step 1: About You

- Age, sex (male / female / other selector), weight (kg), height (cm)
- Grid layout with inputs

### Step 2: Sports & Goals

- Multi‑select sport chips: Running, Cycling, Strength, Swimming, Team Sport
- For each selected sport, choose a goal:
  - 🏃 Maintain Fitness
  - 🏆 Performance
  - 💪 Body Composition
  - 🔄 Return from Layoff

### Step 3: Weekly Schedule

- Day‑of‑week circle toggles (M T W T F S S)
- Minutes‑per‑session slider (15–120 min)
- "Let's Go 🚀" button calls `profile.upsert` mutation and redirects to `/`

---

## 2. Home / Today Screen

The dashboard is a `DashboardHome` component inside a Suspense boundary. The page prefetches `trpc.readiness.getToday` and `trpc.workout.getToday`. The layout below represents the target design.

### Layout

```
┌─────────────────────────────────┐
│  Good morning, [Name]           │
│                                 │
│  ┌───────────────────────────┐  │
│  │   READINESS: 78           │  │
│  │   ████████████░░░  HIGH   │  │
│  │                           │  │
│  │   "HRV strong, but 2     │  │
│  │    hard days — moderate   │  │
│  │    intensity today"       │  │
│  └───────────────────────────┘  │
│                                 │
│  TODAY'S WORKOUT                │
│  ┌───────────────────────────┐  │
│  │  🏃 Tempo Run             │  │
│  │  40–50 min · Zone 3–4    │  │
│  │  Building 5K speed        │  │
│  │                           │  │
│  │  [View Details]           │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌──────────┐ ┌──────────────┐  │
│  │ Too tired│ │ Feeling fresh│  │
│  │   😴     │ │     💪       │  │
│  └──────────┘ └──────────────┘  │
│                                 │
│  QUICK STATS                    │
│  Sleep: 7h 20m  │  HRV: 52ms   │
│  Steps: 8,240   │  Strain: 11.2│
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [⚙️]               │
└─────────────────────────────────┘
```

### Readiness Card

- Large score number with color background:
  - Prime (80+): Green
  - High (60–79): Blue‑green
  - Moderate (40–59): Yellow
  - Low (20–39): Orange
  - Poor (0–19): Red
- Progress bar showing score position
- One‑sentence explanation text

### Adjustment Buttons

- "Too tired?" → down‑shift workout by 1 zone (e.g., tempo → easy run)
- "Feeling fresh?" → up‑shift by 1 zone (e.g., easy → tempo with strides)
- Animates workout card to show new recommendation

---

## 3. Workout Detail Screen

### Layout

```
┌─────────────────────────────────┐
│  ← Back              Share 📤   │
│                                 │
│  🏃 Tempo Run                   │
│  Zone 3–4 · 40–50 min          │
│                                 │
│  WHY THIS TODAY                 │
│  "Your readiness is High (78).  │
│   You've had 2 easy days. Time  │
│   to build 5K speed with        │
│   sustained threshold work."    │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  WORKOUT STRUCTURE              │
│                                 │
│  1. Warm‑up         10 min      │
│     Easy jog, Zone 1–2          │
│     Include dynamic stretches   │
│                                 │
│  2. Main Set         25 min     │
│     Tempo at 4:45/km pace       │
│     Target HR: 155–168 bpm      │
│     Zone 3–4                    │
│                                 │
│  3. Cool‑down        10 min     │
│     Easy jog + walking          │
│     Zone 1                      │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  TARGET METRICS                 │
│  Duration: 40–50 min            │
│  Avg Pace: 4:30–5:00/km        │
│  HR Zone: 3–4 (148–170 bpm)    │
│  Est. Strain: 10–12             │
│                                 │
│  ┌───────────────────────────┐  │
│  │    🎯 Start Workout       │  │
│  └───────────────────────────┘  │
│                                 │
│  [Export to Garmin] (future)    │
└─────────────────────────────────┘
```

---

## 4. History / Trends Screen

### 4.1 Weekly Overview (Default View)

```
┌─────────────────────────────────┐
│  ← Back          7D  28D       │
│                                 │
│  READINESS vs STRAIN            │
│  ┌───────────────────────────┐  │
│  │ 100│·  ·                  │  │
│  │  80│·  ·  ·     ·        │  │
│  │  60│         ·     ·  ·  │  │
│  │  40│              ·      │  │
│  │  20│                     │  │
│  │   0├──┬──┬──┬──┬──┬──┬──│  │
│  │    M  T  W  T  F  S  S  │  │
│  │                          │  │
│  │  ── Readiness  ── Strain │  │
│  └───────────────────────────┘  │
│                                 │
│  THIS WEEK                      │
│  Avg Readiness: 67 (▲ +4)      │
│  Total Strain: 78.4             │
│  Workouts: 5 of 5 planned      │
│                                 │
│  SLEEP TREND                    │
│  Avg: 7h 10m (▼ -20min)        │
│  "Consider earlier bedtime to   │
│   maintain readiness"           │
│                                 │
│  HRV TREND                     │
│  Avg: 48ms (▲ +3ms)            │
│  "Trending up — good recovery"  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [⚙️]               │
└─────────────────────────────────┘
```

### 4.2 Annotations

- Mark hardest workout days with a ⚡ icon
- Mark rest days with 😴
- Show race days with 🏁

---

## 5. Coach Chat Screen

> **Status: Planned — not yet implemented.** The chat router and chat page do not exist in the current codebase.

### Layout

```
┌─────────────────────────────────┐
│  ← Back        Coach Chat 🤖   │
│                                 │
│  ┌───────────────────────────┐  │
│  │ Hi! I'm your coach. Ask  │  │
│  │ me about your training,  │  │
│  │ readiness, or plans.     │  │
│  └───────────────────────────┘  │
│                                 │
│       ┌──────────────────────┐  │
│       │ Why is my readiness  │  │
│       │ lower today?         │  │
│       └──────────────────────┘  │
│                                 │
│  ┌───────────────────────────┐  │
│  │ Your readiness dropped   │  │
│  │ from 78 → 54 today.      │  │
│  │                          │  │
│  │ Main factors:            │  │
│  │ • Sleep: 5h 40m (vs 7h  │  │
│  │   baseline) — ↓ 18 pts  │  │
│  │ • HRV: 38ms (vs 48ms   │  │
│  │   baseline) — ↓ 12 pts  │  │
│  │                          │  │
│  │ 💡 I've adjusted today's │  │
│  │ workout to an easy run.  │  │
│  └───────────────────────────┘  │
│                                 │
│  Quick actions:                 │
│  [What should I do?]            │
│  [Make it harder]               │
│  [Race in N days]               │
│                                 │
│  ┌─────────────────────┐ [Send] │
│  │ Type a message...   │        │
│  └─────────────────────┘        │
└─────────────────────────────────┘
```

### Quick Action Chips

Pre‑built prompts for common intents:
- "What should I do today?"
- "Make today's workout harder/easier"
- "How was my week?"
- "I have a race coming up"

---

## 6. Settings / Profile Screen

- Edit profile (age, weight, height)
- Manage sports & goals
- Update weekly availability
- Garmin connection status & re‑sync
- Notification preferences
- Data & privacy (export, delete)
- About & support

---

## 7. Navigation

### Bottom Tab Bar (BottomNav component)

| Tab | Icon | Route | Screen |
|-----|------|-------|--------|
| Home | 🏠 | `/` | Today / readiness + workout |
| Trends | 📊 | `/trends` | History & charts |
| Settings | ⚙️ | `/settings` | Profile & preferences |

> **Note:** No Chat tab exists yet.

### Key User Journeys

1. **Morning check‑in:** Open app → see readiness → view workout → start training
2. **Post‑workout:** Garmin syncs → strain updates → tomorrow adjusted
3. **Curiosity (planned):** Open chat → "why low?" → understand factors → adjust plans
4. **Race prep (planned):** Chat → "race in 14 days" → see taper plan → follow daily
