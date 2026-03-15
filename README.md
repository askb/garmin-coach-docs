# GarminCoach MVP — WHOOP‑like Coaching App

End‑to‑end MVP spec for a coaching app that ingests Garmin data and outputs
optimal training guidance across sports and goals.

## Documentation Index

| Document | Description |
|----------|-------------|
| [MVP Spec](docs/mvp-spec.md) | Full product specification |
| [Data Model](docs/data-model.md) | Database schema & Garmin data mapping |
| [Readiness Engine](docs/readiness-engine.md) | Readiness score algorithm & strain targets |
| [Coaching Logic](docs/coaching-logic.md) | Workout selection rules & templates |
| [UX Flows](docs/ux-flows.md) | Screens & user journeys |
| [Architecture](docs/architecture.md) | Tech stack, services, deployment |
| [Production Roadmap](docs/production-roadmap.md) | 12‑phase implementation roadmap with code |
| [Dev Environment](docs/dev-environment.md) | Fedora 41 / Linux setup guide |

## Quick Start (Development)

```bash
# Prerequisites: Node.js 22+, pnpm 10+, Docker
pnpm install
docker compose up -d   # Postgres + Redis
pnpm db:push           # Apply schema (drizzle-kit push)
pnpm dev               # Start all dev servers (turbo watch)
pnpm dev:next          # Start web app only
```

## MVP Definition of Done

A single Garmin user can:

1. Connect their Garmin account and see up to 30 days of backfilled data
2. See a daily readiness score (0–100) with color zone and one‑line explanation
3. Get one main workout recommendation per day matching their sport & goal
4. Ask basic questions ("why low?", "harder/easier?") and get data‑backed answers

## License

Proprietary — All rights reserved.
