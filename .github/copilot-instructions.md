# Copilot Instructions for garmin-coach-docs

## Project Overview

This repository contains documentation for the GarminCoach Home Assistant
addon ecosystem — an AI-powered sport scientist for Garmin athletes.

## Documentation Standards

- Use Markdown with proper heading hierarchy (h1 → h2 → h3)
- Include code examples where applicable
- Keep line length under 120 characters where possible
- Use fenced code blocks with language identifiers
- Cross-reference related docs with relative links

## Architecture Context

The GarminCoach system has three main components:

1. **Addon** (`ha-garmin-fitness-coach-addon`): Python sync scripts,
   Docker container, s6-overlay services, PostgreSQL
2. **App** (`ha-garmin-fitness-coach-app`): Next.js frontend, Drizzle ORM,
   tRPC API, AI coaching pipeline
3. **HA Config** (`askb-ha-config`): Home Assistant YAML configuration,
   automations, dashboards

## Key Files

- `garmin-COMPLETE-ARCHITECTURE.md` — Full system architecture
- `docs/` — Additional documentation files
- `README.md` — Repository overview
