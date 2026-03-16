# Development Environment — Fedora 41 Setup Guide

Complete guide for developing GarminCoach v2.0 on Fedora Linux 41.

---

## 1. Compatibility Summary

The entire stack (Next.js website + Expo/React Native Android app + CI pipeline)
works natively on Fedora 41 with no WSL or VM needed.

### Core Stack Compatibility

| Component | Supported | Installation |
|-----------|-----------|-------------|
| Node.js 20–22 LTS | ✅ Excellent | `sudo dnf install nodejs` or use volta/fnm |
| pnpm / npm / yarn | ✅ | `npm install -g pnpm` |
| Next.js 16 / tRPC v11 | ✅ | Pure JS/TS — no issues |
| Drizzle ORM + PostgreSQL | ✅ | `sudo dnf install postgresql-server` or Docker |
| Recharts | ✅ | Pure JS/TS — no issues |
| Turborepo | ✅ | npm package |
| Vitest / Playwright | ✅ | `npx playwright install-deps` for system libs |
| Git | ✅ | Pre‑installed or `sudo dnf install git` |
| VS Code | ✅ | Flatpak or official .rpm |
| Android Studio | ✅ | `sudo dnf install android-studio` or Flatpak |
| EAS Build (cloud) | ✅ 100% | Builds run on Expo servers |
| EAS Build (local) | ✅ | Requires Android SDK/NDK locally |
| iOS builds | ❌ Local | Use EAS cloud builds (macOS runners) |

---

## 2. Base System Setup

### 2.1 — Node.js

```bash
# Option A: System package (Fedora 41 ships Node.js 22.x)
sudo dnf install nodejs npm

# Option B: Volta (recommended for version management)
curl https://get.volta.sh | bash
volta install node@20
volta install pnpm
```

### 2.2 — Global Tools

```bash
npm install -g pnpm eas-cli
```

### 2.3 — Git

```bash
sudo dnf install git
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

### 2.4 — VS Code

```bash
# Option A: Official RPM
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/vscode.repo
sudo dnf install code

# Option B: Flatpak
flatpak install flathub com.visualstudio.code
```

**Recommended extensions:**
- GitHub Copilot + Copilot Chat
- GitLens
- Tailwind CSS IntelliSense
- Drizzle ORM (drizzle-kit)
- ESLint + Prettier

---

## 3. Database Setup

### 3.1 — PostgreSQL via Docker/Podman (Recommended)

```bash
sudo dnf install podman podman-compose
# or: sudo dnf install docker docker-compose

# Use the project's docker-compose.yml
cd garmin-coach
podman-compose up -d    # or: docker compose up -d
```

### 3.2 — PostgreSQL Native (Alternative)

```bash
sudo dnf install postgresql-server postgresql-contrib
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql

sudo -u postgres createuser --interactive  # create your dev user
sudo -u postgres createdb garmincoach
```

### 3.3 — Redis (for caching/queues)

```bash
# Via container (recommended)
podman run -d --name redis -p 6379:6379 redis:7-alpine

# Or native
sudo dnf install redis
sudo systemctl enable --now redis
```

---

## 4. Android Development

### 4.1 — Android Studio

```bash
sudo dnf install android-studio
# Or download from https://developer.android.com/studio
```

After installation:
1. Open Android Studio → SDK Manager
2. Install Android SDK 34 (API 34)
3. Install Android SDK Build‑Tools 34
4. Install Android Emulator + system images

### 4.2 — Environment Variables

```bash
# Add to ~/.bashrc or ~/.zshrc
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
```

### 4.3 — Java (Required for Android Builds)

```bash
# Expo SDK 51+ works best with OpenJDK 17
sudo dnf install java-17-openjdk-devel

# Verify
java -version  # should show 17.x
```

### 4.4 — Create an Emulator

```bash
# Via Android Studio: Tools → Device Manager → Create Virtual Device
# Or via command line:
sdkmanager "system-images;android-34;google_apis;x86_64"
avdmanager create avd -n Pixel_7 -k "system-images;android-34;google_apis;x86_64"
emulator -avd Pixel_7
```

### 4.5 — Permissions Fix (Common Issue)

```bash
# Android SDK permissions
chmod -R 755 ~/Android/Sdk
```

---

## 5. GitHub Copilot CLI

### 5.1 — Installation

```bash
# Option A: Via npm (recommended)
npm install -g @githubnext/github-copilot-cli
github-copilot-cli auth

# Option B: Via GitHub CLI extension
sudo dnf install gh
gh auth login
gh extension install github/gh-copilot
```

Requires an active GitHub Copilot subscription.

### 5.2 — Useful Commands for This Project

```bash
# Generate Drizzle schema from description
copilot suggest "Drizzle ORM schema for user profile with age, weight, sports array"

# Debug build errors
copilot explain "error: Cannot find module 'drizzle-orm'"

# Generate test cases
copilot suggest "vitest tests for readiness score calculation with HRV and sleep inputs"

# Docker/compose help
copilot suggest "docker-compose for postgres 16 and redis with volumes"
```

---

## 6. Playwright Setup (Web E2E Tests)

```bash
# Install browser dependencies for Fedora
npx playwright install
npx playwright install-deps

# This installs Fedora‑compatible system libraries for Chromium, Firefox, WebKit
# May require sudo for system package installation
```

---

## 7. Common Fedora Gotchas & Fixes

### SELinux Interference

```bash
# If PostgreSQL or file watchers fail due to SELinux:
# Temporary (for debugging):
sudo setenforce 0

# Permanent (set proper contexts instead):
sudo setsebool -P httpd_can_network_connect_db 1
```

### Node.js File Watcher Limits

```bash
# If you get "ENOSPC: System limit for number of file watchers reached"
echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Port Conflicts

```bash
# Check if ports are in use
ss -tlnp | grep -E '3000|5432|6379'
```

### Podman vs Docker

```bash
# If using Podman, you may need to enable the Docker socket for compatibility
sudo systemctl enable --now podman.socket
export DOCKER_HOST=unix:///run/podman/podman.sock
```

---

## 8. Quick Start (After Setup)

```bash
# Clone and install
git clone git@github.com:youruser/garmin-coach.git
cd garmin-coach
pnpm install

# Start infrastructure
docker compose up -d    # Postgres + Redis

# Setup database
cp .env.example .env    # Edit with your DATABASE_URL
pnpm db:push            # Apply Drizzle schema
pnpm db:seed            # Load mock data

# Start development
pnpm dev                # Starts web (localhost:3000) + Expo

# Run tests
pnpm test               # Unit + integration
pnpm test:e2e           # Playwright E2E
pnpm lint               # ESLint
pnpm type-check         # TypeScript
```

---

## 9. Recommended System Specs

| Spec | Minimum | Recommended |
|------|---------|-------------|
| RAM | 8 GB | 16 GB (Android emulator is hungry) |
| Storage | 20 GB free | 50 GB (Android SDK + emulator images) |
| CPU | 4 cores | 8 cores (Turborepo parallel builds) |
| Display | 1080p | 1440p+ (emulator + VS Code side‑by‑side) |

---

## 10. Summary

Fedora 41 provides:
- Modern Node.js and toolchain out of the box
- Native Copilot CLI integration
- Rock‑solid terminal tooling (podman, git, gh)
- Full Android local development and testing
- Zero barriers to EAS cloud builds & Vercel deploys

The complete GarminCoach v2.0 platform (backend, engine, website, tests, CI/CD) can be
developed entirely on Fedora 41.
