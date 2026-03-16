# GarminCoach — Home Assistant Addon MVP

## Community Landscape

### What exists today
| Project | What it does | Gap |
|---|---|---|
| [cyberjunky/garmin_connect](https://github.com/cyberjunky/home-assistant-garmin_connect) | HACS integration — exposes Garmin sensors (steps, HR, sleep, stress, body battery) | Raw metrics only — NO coaching, NO training analysis, NO zone tracking |
| [Personal Trainer 2.0](https://community.home-assistant.io/t/make-home-assistant-your-personal-trainer-2-0/611675) | Lovelace dashboard combining Garmin + Strava + scales | Manual setup, no analytics engine, no AI, dashboard-only |
| garmin-fetch-data | Docker container syncing Garmin → InfluxDB | Data pipeline only — no analysis, no UI |

### Our differentiation
**GarminCoach is the ONLY solution that combines:**
1. ✅ Evidence-based sport science engine (ACWR, CTL/ATL/TSB, Z-score readiness)
2. ✅ AI specialist agents (sport scientist, psychologist, nutritionist, recovery)
3. ✅ Zone analytics with polarization tracking (Seiler model)
4. ✅ Race predictions (Riegel formula)
5. ✅ 6+ years of historical trend analysis
6. ✅ Fully local/private (Ollama, no cloud dependency)

**Nothing like this exists as an HA addon.** The community has raw Garmin sensors and manual dashboards — nobody has a coaching/analysis engine.

---

## Option A: iframe Panel (Current Setup)

### Architecture
```
┌──────────────────────────────┐     ┌─────────────────────────┐
│  Home Assistant (HAOS)       │     │  ag15 (192.168.1.77)    │
│  192.168.1.176               │     │                         │
│                              │     │  Next.js :3000          │
│  ┌────────────────────────┐  │     │  PostgreSQL :5432       │
│  │ Sidebar: GarminCoach   │──┼────▶│  Redis :6379            │
│  │ (iframe → :3000)       │  │     │  Ollama :11434          │
│  └────────────────────────┘  │     │                         │
│                              │     │  systemd user service   │
│  panel_iframe in config.yaml │     │  auto-starts on boot    │
└──────────────────────────────┘     └─────────────────────────┘
```

### Setup Steps
1. In HAOS, go to **Settings → Dashboards → Add Dashboard**
2. Choose **"Webpage"** type
3. Set:
   - Title: `GarminCoach`
   - Icon: `mdi:heart-pulse`
   - URL: `http://192.168.1.77:3000`
   - Require admin: No
   - Show in sidebar: Yes

**OR** add to `configuration.yaml`:
```yaml
panel_iframe:
  garmincoach:
    title: "GarminCoach"
    url: "http://192.168.1.77:3000"
    icon: "mdi:heart-pulse"
    require_admin: false
```

### Lovelace Card Alternative
Embed specific pages in any dashboard:
```yaml
type: iframe
url: "http://192.168.1.77:3000/zones"
aspect_ratio: "16:9"
```

---

## Option C: Full HA Addon (Future)

### Addon Architecture
```
┌─────────────────────────────────────────────┐
│  Home Assistant Supervisor                   │
│                                              │
│  ┌─────────────────────────────────────────┐ │
│  │  GarminCoach Addon Container            │ │
│  │                                         │ │
│  │  ┌───────────┐  ┌──────────┐           │ │
│  │  │ Next.js   │  │ Postgres │           │ │
│  │  │ (standalone│  │ (embedded│           │ │
│  │  │  server)  │  │  or ext) │           │ │
│  │  └───────────┘  └──────────┘           │ │
│  │  ┌───────────┐  ┌──────────┐           │ │
│  │  │ Engine    │  │ Redis    │           │ │
│  │  │ (compute) │  │ (cache)  │           │ │
│  │  └───────────┘  └──────────┘           │ │
│  │                                         │ │
│  │  Ingress proxy (no port exposure)       │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  Optional: Ollama addon (separate)           │
└──────────────────────────────────────────────┘
```

### Implementation Phases

#### Phase 1: Standalone Addon (Basic)
- [ ] Create addon repository structure (`config.yaml`, `Dockerfile`, `run.sh`)
- [ ] Containerize: Node.js 22 + Next.js standalone build + embedded SQLite (replace Postgres for simplicity)
- [ ] HA Ingress support (reverse proxy through Supervisor)
- [ ] Config options: Garmin email/password (for garminconnect-python sync)
- [ ] Addon store metadata: icon, logo, description, documentation

**Key decisions:**
- **Database**: SQLite for single-user addon (no Postgres dependency) — Drizzle supports both
- **Data sync**: Embed garminconnect-python for direct Garmin API sync (no InfluxDB needed)
- **Ollama**: Optional — detect if Ollama addon is installed, fallback to rules-based

#### Phase 2: Multi-User + HA Integration
- [ ] HA authentication passthrough (use HA user context)
- [ ] Expose HA sensors: `sensor.garmincoach_readiness`, `sensor.garmincoach_training_status`
- [ ] HA notifications: push alerts for anomalies, deload recommendations
- [ ] Webhook for Garmin Connect IQ (direct watch → addon sync)

#### Phase 3: Community Release
- [ ] HACS repository registration
- [ ] Documentation: installation guide, screenshots, FAQ
- [ ] Multi-device support (Garmin, Wahoo, Polar via FIT file import)
- [ ] Translations (i18n)
- [ ] Community beta testing

### Addon Repository Structure
```
garmincoach-addon/
├── config.yaml              # Addon metadata
├── Dockerfile               # Multi-stage build
├── run.sh                   # Entrypoint with bashio
├── DOCS.md                  # Shows in addon info panel
├── CHANGELOG.md
├── icon.png                 # 256x256 addon icon
├── logo.png                 # 256x256 addon logo
├── translations/
│   └── en.yaml
├── apparmor.txt             # Security profile
└── build.yaml               # Multi-arch (amd64, aarch64)
```

### config.yaml (Addon Manifest)
```yaml
name: "GarminCoach"
description: "AI-powered sport scientist — training analysis, coaching, and recovery optimization from your Garmin data"
version: "1.0.0"
slug: "garmincoach"
url: "https://github.com/askb/garmincoach-addon"
arch:
  - amd64
  - aarch64
homeassistant_api: true
ingress: true
ingress_port: 3000
panel_icon: "mdi:heart-pulse"
panel_title: "GarminCoach"
map:
  - addon_config:rw
  - share:rw
ports:
  3000/tcp: null  # null = ingress only, no host port
options:
  garmin_email: ""
  garmin_password: ""
  ollama_url: ""
  language: "en"
schema:
  garmin_email: email
  garmin_password: password
  ollama_url: "url?"
  language: "str?"
startup: "application"
init: false
```

### Dockerfile (Multi-Stage)
```dockerfile
# Stage 1: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY . .
RUN npm install -g pnpm && pnpm install --frozen-lockfile
RUN pnpm --filter @acme/nextjs build
# Standalone output mode — single directory with everything

# Stage 2: Runtime
FROM node:22-alpine
RUN apk add --no-cache python3 py3-pip sqlite
WORKDIR /app
COPY --from=builder /app/apps/nextjs/.next/standalone ./
COPY --from=builder /app/apps/nextjs/.next/static ./.next/static
COPY --from=builder /app/apps/nextjs/public ./public

# Garmin sync (Python)
RUN pip3 install garminconnect

COPY run.sh /run.sh
RUN chmod +x /run.sh
CMD ["/run.sh"]
```

### Estimated Resource Usage (as HA Addon)
| Component | RAM | CPU | Disk |
|---|---|---|---|
| Next.js standalone | ~80MB | <1% idle | ~50MB |
| SQLite | ~5MB | <1% | ~20MB data |
| Garmin sync (cron) | ~30MB (peak) | burst | minimal |
| **Total** | **~115MB** | **<2% idle** | **~70MB** |

Compared to other addons: lighter than Grafana (200MB+), similar to Node-RED (~100MB).

### Migration Path (Current → Addon)
1. Replace PostgreSQL with SQLite/better-sqlite3 (Drizzle adapter swap)
2. Next.js `output: "standalone"` build (already supported)
3. Embed garminconnect-python for direct sync (replace InfluxDB pipeline)
4. Strip Redis (use in-memory cache for single-user)
5. HA Ingress integration (auth header passthrough)

### Community Value Assessment
- **Target audience**: ~500K+ Garmin users in HA community (based on cyberjunky integration downloads)
- **Unique value**: Only sport science + AI coaching addon for HA
- **Comparable addons**: ESPHome (~1M installs), Node-RED (~500K), Grafana (~200K)
- **Realistic reach**: 5K-20K installs in first year if quality is high
