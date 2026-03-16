# UX Flows & Screens — GarminCoach v2.1

## Screen Map

```
/              → Home / Today (readiness + workout + quick stats)
/onboarding    → 3-step setup flow
/trends        → Advanced Trends (multi-metric overlay, regression, correlations)
/training      → Training Load (CTL/ATL/TSB, ACWR, load focus, status)
/zones         → Zone Analytics (7-chart dashboard, polarization, efficiency)
/sleep         → Sleep Dashboard (stages, score, debt, coach, timing)
/settings      → Profile, Garmin connection, preferences
/workout/[id]  → Workout Detail (structure, targets, explanation)
/coach         → AI Coach (4 specialist agents, data-driven chat)
```

---

## Navigation — 6-Tab Bottom Bar

| Tab | Icon | Route | Screen |
|-----|------|-------|--------|
| Today | 🏠 | `/` | Home — readiness + workout |
| Trends | 📊 | `/trends` | Advanced Trends |
| Training | 🏋️ | `/training` | Training Load |
| Zones | 📈 | `/zones` | Zone Analytics |
| Sleep | 🌙 | `/sleep` | Sleep Dashboard |
| Settings | ⚙️ | `/settings` | Settings & Profile |

---

## 1. Onboarding Flow (3 Steps)

Progress bar shows 3 segments. Minimal friction — get to the dashboard fast.

### Step 1: About You

- Age, sex (male / female / other selector), weight (kg), height (cm)
- Grid layout with inputs
- Experience level selector: Beginner / Intermediate / Advanced

### Step 2: Sports & Goals

- Multi-select sport chips: Running, Cycling, Strength, Swimming, Team Sport
- For each selected sport, choose a goal:
  - 🏃 Maintain Fitness
  - 🏆 Performance
  - 💪 Body Composition
  - 🔄 Return from Layoff

### Step 3: Weekly Schedule

- Day-of-week circle toggles (M T W T F S S)
- Minutes-per-session slider (15–120 min)
- "Let's Go 🚀" button calls `profile.upsert` and redirects to `/`

---

## 2. Home / Today Screen

The primary daily touchpoint. Prefetches `readiness.getToday` and `workout.getToday`.

### Layout

```
┌─────────────────────────────────┐
│  Good morning, [Name]           │
│                                 │
│  ┌───────────────────────────┐  │
│  │   READINESS: 78           │  │
│  │   ████████████░░░  GOOD   │  │
│  │   Confidence: High        │  │
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
│  │  Est. Recovery: 24–36h   │  │
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
│  RHR: 58 bpm    │  ACWR: 1.05  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Interactions

- **Readiness card:** Large score with zone color background and confidence badge
- **Workout card:** Tap → Workout Detail page
- **Adjustment buttons:** "Too tired" / "Feeling fresh" → re-generate workout with ±1 zone shift
- **Quick stats:** Tap any stat → navigates to relevant dashboard

---

## 3. Advanced Trends Page

Multi-metric analytics dashboard with Recharts visualizations.

### Layout

```
┌─────────────────────────────────┐
│  Advanced Trends         7D 28D │
│                                 │
│  METRIC OVERLAY CHART           │
│  ┌───────────────────────────┐  │
│  │ [Multi-line Recharts]     │  │
│  │ Readiness ── Strain ──   │  │
│  │ HRV ── Sleep ── RHR      │  │
│  │                           │  │
│  │ Toggle metrics on/off     │  │
│  └───────────────────────────┘  │
│                                 │
│  TREND ANALYSIS                 │
│  ┌───────────────────────────┐  │
│  │ Readiness: ↑ Improving    │  │
│  │   +3.2/week (p=0.02)     │  │
│  │ HRV: → Stable             │  │
│  │   +0.1/week (p=0.81)     │  │
│  │ Sleep: ↓ Declining        │  │
│  │   −12 min/week (p=0.04)  │  │
│  └───────────────────────────┘  │
│                                 │
│  CORRELATIONS                   │
│  ┌───────────────────────────┐  │
│  │ Sleep → Readiness: r=0.72│  │
│  │ HRV → Readiness:   r=0.68│  │
│  │ Load → Readiness:  r=−0.45│ │
│  │ Sleep → HRV:       r=0.61│  │
│  │ Stress → Readiness: r=−0.38│ │
│  │ RHR → Readiness:  r=−0.31│  │
│  └───────────────────────────┘  │
│                                 │
│  NOTABLE CHANGES                │
│  ┌───────────────────────────┐  │
│  │ ⚠️ HRV dropped 18% below │  │
│  │   baseline on Thu         │  │
│  │ ✅ Sleep quality improved  │  │
│  │   since bedtime adjustment│  │
│  └───────────────────────────┘  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Features

- **Metric overlay:** Select which metrics to display on a shared time axis
- **Period toggle:** 7-day and 28-day views
- **Trend regression:** Linear regression with slope, direction, significance
- **Correlation table:** Pearson r for 6 standard metric pairs
- **Notable changes:** Annotated markers for significant deviations

---

## 4. Training Load Page

Fitness-fatigue model visualization with load management tools.

### Layout

```
┌─────────────────────────────────┐
│  Training Load          28D 90D │
│                                 │
│  CTL / ATL / TSB CHART          │
│  ┌───────────────────────────┐  │
│  │ [Recharts line chart]     │  │
│  │ CTL (fitness) ──          │  │
│  │ ATL (fatigue) ──          │  │
│  │ TSB (form) ── (area fill) │  │
│  └───────────────────────────┘  │
│                                 │
│  TRAINING STATUS                │
│  ┌───────────────────────────┐  │
│  │  🟢 PRODUCTIVE            │  │
│  │  Fitness improving with    │  │
│  │  appropriate load balance  │  │
│  └───────────────────────────┘  │
│                                 │
│  ACWR GAUGE                     │
│  ┌───────────────────────────┐  │
│  │    [Gauge / speedometer]   │  │
│  │         1.12               │  │
│  │    ◄─────●──────►          │  │
│  │  0.8   sweet   1.3   1.5  │  │
│  │  under  spot  caution dngr │  │
│  └───────────────────────────┘  │
│                                 │
│  LOAD FOCUS (Last 7 Days)       │
│  ┌───────────────────────────┐  │
│  │  [Pie/donut chart]         │  │
│  │  Aerobic: 65%              │  │
│  │  Anaerobic: 20%            │  │
│  │  Mixed: 15%                │  │
│  └───────────────────────────┘  │
│                                 │
│  RECOVERY                       │
│  ┌───────────────────────────┐  │
│  │  Est. recovery: 18h left   │  │
│  │  Next hard session: OK     │  │
│  │  Ramp rate: 4.2 TSS/day   │  │
│  └───────────────────────────┘  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Features

- **CTL/ATL/TSB chart:** Dual-axis line chart with TSB as filled area
- **Training status badge:** One of 6 states with explanation
- **ACWR gauge:** Visual gauge showing current position in risk zones
- **Load focus:** Pie/donut chart of aerobic/anaerobic/mixed distribution
- **Recovery panel:** Estimated recovery remaining, ramp rate warning
- **Period toggle:** 28-day and 90-day views

---

## 4a. Zone Analytics Page (NEW)

HR zone distribution dashboard with 7 chart sections and sport filtering.

### Layout

```
┌─────────────────────────────────┐
│  Zone Analytics     4W 8W 12W   │
│                   [All Sports ▼] │
│                                 │
│  WEEKLY ZONE DISTRIBUTION       │
│  ┌───────────────────────────┐  │
│  │ [Stacked bar chart]       │  │
│  │ Zone 1–5 minutes/week    │  │
│  │ Each bar = 1 ISO week     │  │
│  └───────────────────────────┘  │
│                                 │
│  POLARIZATION INDEX             │
│  ┌───────────────────────────┐  │
│  │ [Composed line chart]     │  │
│  │ PI = 1.87 (Polarized ✓)  │  │
│  │ Easy: 78% | Mod: 5% |    │  │
│  │ Hard: 17%                 │  │
│  └───────────────────────────┘  │
│                                 │
│  ZONE TRENDS (Monthly)          │
│  ┌───────────────────────────┐  │
│  │ [100% stacked area chart] │  │
│  │ Zone % breakdown by month │  │
│  └───────────────────────────┘  │
│                                 │
│  EFFICIENCY TREND               │
│  ┌───────────────────────────┐  │
│  │ [Scatter chart]            │  │
│  │ Pace/HR efficiency index  │  │
│  │ per activity over time     │  │
│  └───────────────────────────┘  │
│                                 │
│  ACTIVITY CALENDAR              │
│  ┌───────────────────────────┐  │
│  │ [CSS grid heatmap]        │  │
│  │ Daily activity intensity   │  │
│  │ GitHub-style calendar      │  │
│  └───────────────────────────┘  │
│                                 │
│  VOLUME BY WEEK                 │
│  ┌───────────────────────────┐  │
│  │ [Stacked bar chart]       │  │
│  │ Weekly minutes by sport   │  │
│  └───────────────────────────┘  │
│                                 │
│  PEAK PERFORMANCES              │
│  ┌───────────────────────────┐  │
│  │ Monthly bests:             │  │
│  │ Best pace, longest run,   │  │
│  │ longest duration, max HR  │  │
│  └───────────────────────────┘  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Features

- **Period selector:** 4-week, 8-week, 12-week views
- **Sport filter:** Filter all charts by sport type (running, cycling, etc.)
- **Polarization tracking:** Seiler's 80/20 model with PI formula
- **Efficiency scatter:** Speed/HR ratio trend for aerobic fitness tracking
- **Calendar heatmap:** GitHub-style CSS grid showing daily activity intensity
- **7 tRPC endpoints:** Each chart section backed by its own optimized query

---

## 4b. AI Coach Page (NEW)

4 specialist AI agents powered by local Ollama inference with real-time data context.

### Layout

```
┌─────────────────────────────────┐
│  AI Coach                       │
│                                 │
│  ┌───┬───┬───┬───┐             │
│  │🔬 │🧠 │🥗 │💤 │  Agent tabs  │
│  │Sci│Psy│Nut│Rec│             │
│  └───┴───┴───┴───┘             │
│                                 │
│  QUICK PROMPTS                  │
│  [What should I train today?]   │
│  [Why is my readiness low?]     │
│  [How's my training balance?]   │
│                                 │
│  CHAT                           │
│  ┌───────────────────────────┐  │
│  │ 🤖 Sport Scientist:       │  │
│  │ Based on your last 14 days│  │
│  │ your polarization index   │  │
│  │ is 1.87 — good 80/20...  │  │
│  │                           │  │
│  │ 👤 You:                    │  │
│  │ Should I do intervals     │  │
│  │ tomorrow?                 │  │
│  │                           │  │
│  │ 🤖 [typing indicator ···] │  │
│  └───────────────────────────┘  │
│                                 │
│  ┌───────────────────────────┐  │
│  │ Type a message...    [Send]│  │
│  └───────────────────────────┘  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Features

- **Agent selector tabs:** Switch between 4 specialist personas
- **Quick prompt chips:** Pre-built questions for common queries
- **Typing indicator:** Shows while Ollama generates response
- **Markdown rendering:** Responses rendered with headers, lists, bold
- **Data-driven context:** Each message includes last 14 days of real metrics
- **Fallback:** Template responses if Ollama is unavailable

---

## 5. Sleep Dashboard

Comprehensive sleep analytics and coaching.

### Layout

```
┌─────────────────────────────────┐
│  Sleep Dashboard         7D 28D │
│                                 │
│  LAST NIGHT                     │
│  ┌───────────────────────────┐  │
│  │  Sleep Score: 82          │  │
│  │  Duration: 7h 45m         │  │
│  │  Efficiency: 91%          │  │
│  └───────────────────────────┘  │
│                                 │
│  SLEEP STAGES                   │
│  ┌───────────────────────────┐  │
│  │  [Stacked bar chart]      │  │
│  │  Deep ███ 1h 20m (18%)   │  │
│  │  REM  ███ 1h 45m (23%)   │  │
│  │  Light ██ 4h 10m (54%)   │  │
│  │  Awake █  30m (6%)       │  │
│  └───────────────────────────┘  │
│                                 │
│  SLEEP DEBT                     │
│  ┌───────────────────────────┐  │
│  │  7-Day Debt: 2h 15m       │  │
│  │  Status: ✅ Manageable    │  │
│  │  [Bar chart: daily debt]  │  │
│  └───────────────────────────┘  │
│                                 │
│  SLEEP COACH                    │
│  ┌───────────────────────────┐  │
│  │  💡 Your sleep need: 8.5h │  │
│  │  (base 7.5h + athlete +1h)│  │
│  │                           │  │
│  │  Recommended bedtime:     │  │
│  │  10:15 PM (for 6:30 AM   │  │
│  │  wake time)               │  │
│  │                           │  │
│  │  Tip: Your deep sleep %   │  │
│  │  improves when bedtime is │  │
│  │  before 10:30 PM          │  │
│  └───────────────────────────┘  │
│                                 │
│  TIMING ANALYSIS                │
│  ┌───────────────────────────┐  │
│  │  Avg bedtime: 10:42 PM    │  │
│  │  Avg wake time: 6:28 AM   │  │
│  │  Consistency: 82%          │  │
│  └───────────────────────────┘  │
│                                 │
│  ─────────────────────────────  │
│  [🏠] [📊] [🏋️] [📈] [🌙] [⚙️]│
└─────────────────────────────────┘
```

### Key Features

- **Sleep score:** Last night's score with trend indicator
- **Sleep stages chart:** Stacked bar showing deep/REM/light/awake distribution
- **Sleep debt tracker:** Rolling 7-day cumulative debt with severity badge
- **Sleep coach:** Personalized sleep need, bedtime recommendation, tips
- **Timing analysis:** Bedtime/wake time consistency tracking

---

## 6. Settings Page

- **Profile:** Edit age, weight, height, experience level
- **Sports & Goals:** Add/remove sports, change goals
- **Weekly Schedule:** Update available days and time per session
- **Garmin Connection:** Status, last sync time, reconnect, disconnect
- **Preferences:** Units (metric/imperial), notification settings
- **Data & Privacy:** Export data, delete account
- **About:** Version, support, privacy policy

---

## 7. Workout Detail Page

### Layout

```
┌─────────────────────────────────┐
│  ← Back              Share 📤   │
│                                 │
│  🏃 Tempo Run                   │
│  Zone 3–4 · 40–50 min          │
│  Est. Strain: 10–12             │
│                                 │
│  WHY THIS TODAY                 │
│  "Your readiness is Good (78).  │
│   You've had 2 easy days. Time  │
│   to build 5K speed with        │
│   sustained threshold work.     │
│   Training status: Productive." │
│                                 │
│  ─────────────────────────────  │
│                                 │
│  WORKOUT STRUCTURE              │
│                                 │
│  1. Warm-up         10 min      │
│     Easy jog, Zone 1–2          │
│     Include dynamic stretches   │
│                                 │
│  2. Main Set         25 min     │
│     Tempo at 4:45/km pace       │
│     Target HR: 155–168 bpm      │
│     Zone 3–4                    │
│                                 │
│  3. Cool-down        10 min     │
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
│  Est. Recovery: 24–36h          │
│                                 │
│  ┌───────────────────────────┐  │
│  │    🎯 Start Workout       │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

---

## 8. Key User Journeys

### Journey 1: Morning Check-In (Primary)
1. Open app → see readiness score + zone
2. Review today's workout recommendation
3. Check quick stats (sleep, HRV, strain)
4. Tap "View Details" → review workout structure
5. Go train

### Journey 2: Post-Workout Review
1. Garmin syncs → strain updates automatically
2. Open app → see updated training load
3. Check ACWR gauge on Training page
4. View recovery time estimate
5. Tomorrow's workout auto-adjusts

### Journey 3: Weekly Analysis
1. Open Trends tab → review 7-day metric overlay
2. Check trend regression (improving/declining/stable)
3. Review correlations (what's driving readiness?)
4. Note any notable changes flagged by the system

### Journey 4: Sleep Optimization
1. Open Sleep tab → review last night's score/stages
2. Check 7-day sleep debt status
3. Read sleep coach recommendations
4. Adjust bedtime based on advice
5. Track improvement over time

### Journey 5: Training Load Management
1. Open Training tab → review CTL/ATL/TSB chart
2. Check training status badge (productive/overreaching/etc.)
3. Monitor ACWR gauge — stay in sweet spot
4. Review load focus distribution
5. Adjust plan if overreaching detected

### Journey 6: Zone Analysis & Polarization
1. Open Zones tab → review weekly zone distribution
2. Check polarization index (target: 80/20 split)
3. Review zone trends over months
4. Check efficiency scatter for aerobic fitness progression
5. Browse activity calendar heatmap for consistency patterns
6. Filter by sport to isolate running vs cycling zones

### Journey 7: AI Coaching Consultation
1. Open AI Coach → select specialist agent (e.g., Sport Scientist)
2. Tap quick prompt or type custom question
3. Agent responds with data-backed advice (uses last 14 days of metrics)
4. Switch to Recovery Specialist for sleep/deload guidance
5. Responses rendered in markdown with actionable recommendations

---

## 9. Charts Library (Recharts)

| Chart | Page | Type | Data Source |
|-------|------|------|-------------|
| Multi-metric overlay | Trends | Line (multi-series) | trends.getMultiMetric |
| CTL/ATL/TSB | Training | Line + Area | trends.getCTLATLTSB |
| ACWR gauge | Training | Gauge/radial | trends.getACWR |
| Load focus | Training | Pie/Donut | analytics.getLoadFocus |
| Sleep stages | Sleep | Stacked Bar | sleep.getDashboard |
| Sleep debt | Sleep | Bar | sleep.getDebt |
| Readiness history | Today | Sparkline | readiness.getHistory |
| Zone distribution | Zones | Stacked Bar | zones.getWeeklyZoneDistribution |
| Polarization index | Zones | Composed (Line + Bar) | zones.getPolarizationIndex |
| Zone trends | Zones | 100% Stacked Area | zones.getZoneTrends |
| Efficiency trend | Zones | Scatter | zones.getEfficiencyTrend |
| Activity calendar | Zones | CSS Grid Heatmap | zones.getActivityCalendar |
| Volume by week | Zones | Stacked Bar | zones.getVolumeByWeek |
| Peak performances | Zones | Table / Cards | zones.getPeakPerformances |
