# Agent Guidelines for garmin-coach-docs

## Repository Purpose

Documentation repository for the GarminCoach HA addon ecosystem.
Contains architecture docs, API references, and user guides.

## Commit Format

```
type(scope): description

Signed-off-by: Anil Belur <askb23@gmail.com>
Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

**Allowed types** (case-insensitive):
`fix`, `feat`, `chore`, `docs`, `style`, `refactor`

**Use lowercase** for PR titles and commit messages.

## Rules

1. All commits must include `Signed-off-by` (DCO compliance)
2. All AI-assisted commits must include `Co-authored-by` trailer
3. Markdown files must pass markdownlint
4. Keep documentation accurate and up-to-date with the codebase
5. Do not include secrets, credentials, or private infrastructure details

## Related Repositories

- **Addon**: `askb/ha-garmin-fitness-coach-addon` (Python sync, Docker, s6)
- **App**: `askb/ha-garmin-fitness-coach-app` (Next.js, Drizzle, tRPC)
- **HA Config**: `askb/askb-ha-config` (Home Assistant configuration)
