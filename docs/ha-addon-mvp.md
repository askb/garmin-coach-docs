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

## Deployed Architecture (v2.0)

### Addon Architecture
```
┌─────────────────────────────────────────────────────────────┐
│  Home Assistant Supervisor (HAOS)                           │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  GarminCoach Addon Container (aarch64)                │  │
│  │                                                       │  │
│  │  ┌─────────────────┐  ┌──────────────────────────┐   │  │
│  │  │ Ingress Proxy   │  │ Next.js 16 Standalone    │   │  │
│  │  │ :3000 (public)  │─▶│ :3001 (internal)         │   │  │
│  │  │                 │  │                          │   │  │
│  │  │ Rewrites ALL    │  │ T3 Turbo: tRPC v11      │   │  │
│  │  │ /_next/ paths   │  │ + Drizzle ORM + RSC     │   │  │
│  │  └─────────────────┘  └──────────────────────────┘   │  │
│  │                                                       │  │
│  │  ┌─────────────────┐  ┌──────────────────────────┐   │  │
│  │  │ PostgreSQL 16   │  │ Garmin Auth Server       │   │  │
│  │  │ :5432 (embedded)│  │ Flask :8099 (internal)   │   │  │
│  │  └─────────────────┘  └──────────────────────────┘   │  │
│  │                                                       │  │
│  │  ┌─────────────────┐  ┌──────────────────────────┐   │  │
│  │  │ Garmin Sync     │  │ s6-overlay service mgr   │   │  │
│  │  │ (cron/Python)   │  │ (process supervision)    │   │  │
│  │  └─────────────────┘  └──────────────────────────┘   │  │
│  │                                                       │  │
│  │  /data/garmin-tokens/ (persistent across restarts)    │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  Optional: Ollama addon (separate) for AI coaching          │
│  Default: HA Conversation API for AI (OpenClaw/Claude)      │
└─────────────────────────────────────────────────────────────┘
```

### HA Ingress Proxy (Critical Component)

The proxy (`ingress-proxy.js`) sits between HA ingress and Next.js, solving the
path rewriting problem. HA serves the addon at `/api/hassio_ingress/<token>/`
but Next.js emits absolute `/_next/` paths — the proxy rewrites them.

**What it rewrites:**
| Response Type | Rewriting |
|---|---|
| HTML (`text/html`) | `/_next/` paths + `href`/`src`/`action` attributes + `<meta>` tag injection |
| JavaScript (ALL) | `/_next/` in imports, chunk loaders, turbopack runtime |
| CSS | `url(/_next/...)` for fonts and assets |
| RSC flight (`text/x-component`, `text/plain`) | Module references for client-side navigation |
| JSON API | Any `/_next/` references |
| Binary (fonts, images) | Passthrough — no rewriting |

The universal rewrite is: `replaceAll("/_next/", ingressPath + "/_next/")`

---

## Garmin Connect Authentication

### Known Issues (2025-2026)

The `garminconnect` / `garth` Python libraries have a **known upstream bug** with
MFA-enabled Garmin accounts. Garmin changed their SSO backend, breaking automated
MFA login flows.

**Symptoms:**
- `401 Client Error: Unauthorized` on login
- `OAuth1 token is required for OAuth2 refresh`
- MFA code prompt never reached

**Tracked in:**
- [cyberjunky/python-garminconnect #312](https://github.com/cyberjunky/python-garminconnect/issues/312)
- [cyberjunky/python-garminconnect #235](https://github.com/cyberjunky/python-garminconnect/issues/235)
- [matin/garth #137](https://github.com/matin/garth/issues/137)

### Recommended Workaround: Local Token Generation

Since the addon runs headless and can't handle interactive MFA, generate tokens
on your laptop where MFA works, then import them into the addon.

**Step 1: Generate tokens locally**
```bash
cd ~/git/garmincoach-addon
python3 scripts/generate-garmin-tokens.py
```

The script will:
1. Prompt for your Garmin email and password
2. Handle MFA interactively (check your phone for the code)
3. Save OAuth1 + OAuth2 tokens to `/tmp/garmin-tokens/`
4. Offer to deploy tokens to HAOS automatically

**Step 2: Import tokens into the addon**

Option A — Automatic (via the script's deploy step):
The script copies tokens to HAOS and calls the addon's `/auth/import-tokens` API.

Option B — Manual:
```bash
# Copy token files to HAOS
cat /tmp/garmin-tokens/oauth1_token.json | \
  ssh -p 22222 hassio@192.168.1.176 "cat > /tmp/oauth1_token.json"
cat /tmp/garmin-tokens/oauth2_token.json | \
  ssh -p 22222 hassio@192.168.1.176 "cat > /tmp/oauth2_token.json"

# Import via addon API (from HAOS SSH)
ssh -p 22222 hassio@192.168.1.176
curl -X POST -H 'Content-Type: application/json' \
  -d "$(python3 -c 'import json; o1=json.load(open("/tmp/oauth1_token.json")); o2=json.load(open("/tmp/oauth2_token.json")); print(json.dumps({"oauth1_token":o1,"oauth2_token":o2}))')" \
  'http://ecfdb23d-garmincoach:3000/api/garmin/auth-import'
```

**Token Lifecycle:**
- Tokens are valid for weeks/months until Garmin expires them
- The auth server auto-refreshes OAuth2 using the saved OAuth1 token
- Tokens persist in `/data/garmin-tokens/` across addon restarts
- If tokens expire, re-run the generate script

### Auth Server Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/api/garmin/auth` | Check connection status |
| POST | `/api/garmin/auth` | Login (email + password) or MFA (code) |
| PUT | `/api/garmin/auth` | Import pre-generated tokens |
| DELETE | `/api/garmin/auth` | Disconnect / remove tokens |

### In-App Login (When MFA is Not Required)

If your Garmin account does **not** have MFA enabled, you can log in directly
from the Settings page in the addon UI:

1. Navigate to **Settings** in the GarminCoach sidebar
2. Enter your Garmin email and password
3. Click **Connect**

---

## Addon Configuration

### config.yaml Options
```yaml
options:
  garmin_email: ""          # Garmin Connect email (optional if using token import)
  garmin_password: ""       # Garmin Connect password (optional if using token import)
  ai_backend: "ha_conversation"  # ha_conversation | ollama | none
  ollama_url: ""            # e.g., http://homeassistant.local:11434
  sync_interval_minutes: 60 # How often to pull Garmin data
```

### Addon Repository Structure (Actual)
```
garmincoach-addon/
├── garmincoach/
│   ├── config.yaml              # Addon metadata + schema
│   ├── Dockerfile               # Multi-stage: node:22-alpine → HA aarch64-base
│   ├── build.yaml               # Arch-specific build config
│   ├── CHANGELOG.md
│   ├── DOCS.md
│   └── rootfs/
│       ├── app/
│       │   ├── ingress-proxy.js         # HA ingress path rewriter
│       │   └── scripts/
│       │       ├── garmin-auth-server.py # Flask auth microservice
│       │       └── garmin-sync.py       # Periodic Garmin data sync
│       └── etc/
│           └── s6-overlay/s6-rc.d/      # Service definitions
│               ├── garmincoach/run      # Main service (PG + Next.js + proxy)
│               ├── garmin-auth/run      # Auth server service
│               └── postgresql/run       # PostgreSQL service
├── repository.yaml              # HA addon repository metadata
├── scripts/
│   ├── build-local.sh           # Local Docker build helper
│   └── generate-garmin-tokens.py # Token generation for MFA accounts
└── tests/
    └── unit/
        └── test_ingress_proxy.js # 20 E2E proxy tests
```

---

## Resource Usage (Measured on aarch64)
| Component | RAM | CPU | Disk |
|---|---|---|---|
| Next.js 16 standalone | ~80MB | <1% idle | ~50MB |
| PostgreSQL 16 | ~30MB | <1% idle | ~20MB data |
| Garmin sync (cron) | ~30MB (peak) | burst | minimal |
| Flask auth server | ~15MB | <1% | minimal |
| Ingress proxy | ~10MB | <1% | minimal |
| **Total** | **~165MB** | **<2% idle** | **~70MB** |

---

## Future Phases

### Phase 2: HA Integration
- [ ] HA authentication passthrough (use HA user context)
- [ ] Expose HA sensors: `sensor.garmincoach_readiness`, `sensor.garmincoach_training_status`
- [ ] HA notifications: push alerts for anomalies, deload recommendations
- [ ] Webhook for Garmin Connect IQ (direct watch → addon sync)

### Phase 3: Community Release
- [ ] HACS repository registration
- [ ] Addon store icons (icon.png + logo.png, 256×256)
- [ ] Documentation: installation guide, screenshots, FAQ
- [ ] Multi-device support (Garmin, Wahoo, Polar via FIT file import)
- [ ] Translations (i18n)
- [ ] Community beta testing

### Community Value Assessment
- **Target audience**: ~500K+ Garmin users in HA community
- **Unique value**: Only sport science + AI coaching addon for HA
- **Realistic reach**: 5K-20K installs in first year if quality is high
