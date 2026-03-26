# 007: Google Fit Research

## Status: Draft

## Problem Statement

There is interest in adding Google Fit as a data source, but Google has
deprecated the Google Fit REST API in favor of Health Connect (an on-device
Android API). We need to evaluate whether a Google Fit / Health Connect
integration is feasible and what the technical approach would be.

## Requirements

- [ ] Research current status of Google Fit REST API deprecation
- [ ] Evaluate Health Connect (Android) as an alternative data source
- [ ] Identify what data types are available via each path
- [ ] Assess feasibility for a Home Assistant addon / web app architecture
- [ ] Document a clear recommendation: build, wait, or skip

## Acceptance Criteria

- [ ] A research document covering API status, available data, and limitations
- [ ] A feasibility assessment with pros/cons for each integration path
- [ ] A clear recommendation with rationale
- [ ] Identified blockers or prerequisites if recommending to proceed

## Out of Scope

- Building any integration (this is research only)
- Evaluating Apple Health (separate research effort)
- Pricing or commercial considerations

## Technical Context

- Google Fit REST API was deprecated in 2024 with shutdown planned
- Health Connect is an on-device Android API (no REST endpoint)
- Our architecture is a web app with a Home Assistant addon — no native
  Android component
- Alternative paths might include: Health Connect via a companion Android
  app, third-party sync services, or direct export/import
