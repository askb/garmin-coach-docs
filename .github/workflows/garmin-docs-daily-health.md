---
on:
  schedule: daily on weekdays

permissions:
  contents: read
  issues: read
  pull-requests: read

network: defaults

safe-outputs:
  create-issue:
    title-prefix: "[daily-health] "
    labels: [report, daily-health]
    close-older-issues: true
    max: 1
---

# Daily Documentation Health Report

Generate a daily health report for the Garmin Coach documentation repository.

## Context

This repository contains documentation for the GarminCoach HA addon ecosystem
spanning 4 repos: addon (Python), app (Next.js), HA config, and this docs repo.

## Instructions

Create a concise daily health report as a GitHub issue covering:

### 1. Repository Activity (last 24h)
- Recent commits and what changed
- Open pull requests needing review
- Any CI/CD failures in recent workflow runs

### 2. Documentation Inventory
- Count of markdown files in docs/ directory
- Count of architecture/design docs
- Any files modified in the last 7 days

### 3. Potential Issues
- Check for TODO/FIXME/PLACEHOLDER markers in docs
- Look for broken relative links between docs
- Check for outdated date references
- Note any docs that reference deprecated features

### 4. Recommendations
- Suggest docs that may need updating based on recent changes
- Flag any gaps in documentation coverage
- Note cross-repo documentation inconsistencies

### Format
Use clear headings, bullet points, and emoji status indicators:
- ✅ Healthy
- ⚠️ Needs attention
- ❌ Action required

Keep the report under 400 words. Focus on actionable items only.
