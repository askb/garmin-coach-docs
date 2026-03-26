# Contributing to garmin-coach-docs

Thank you for your interest in improving the GarminCoach documentation!
This guide explains how to set up the project locally, follow our style
guidelines, and submit your changes.

## Table of Contents

- [Related Repositories](#related-repositories)
- [Clone and Local Setup](#clone-and-local-setup)
- [Pre-commit Hook Setup](#pre-commit-hook-setup)
- [Documentation Style Guidelines](#documentation-style-guidelines)
- [Submitting Changes via PR](#submitting-changes-via-pr)

---

## Related Repositories

The GarminCoach ecosystem spans four repositories:

| Repository | Description |
|------------|-------------|
| [askb/garmin-coach-docs](https://github.com/askb/garmin-coach-docs) | This repo — architecture docs, API references, and user guides |
| [askb/ha-garmin-fitness-coach-addon](https://github.com/askb/ha-garmin-fitness-coach-addon) | Python sync scripts, Docker container, s6-overlay services, PostgreSQL |
| [askb/ha-garmin-fitness-coach-app](https://github.com/askb/ha-garmin-fitness-coach-app) | Next.js frontend, Drizzle ORM, tRPC API, AI coaching pipeline |
| [askb/askb-ha-config](https://github.com/askb/askb-ha-config) | Home Assistant YAML configuration, automations, and dashboards |

---

## Clone and Local Setup

### Prerequisites

- [Git](https://git-scm.com/) 2.x or later
- [Python](https://www.python.org/) 3.9 or later (for pre-commit)
- [Node.js](https://nodejs.org/) 20 LTS or later (for markdownlint-cli, optional)
- [pip](https://pip.pypa.io/) or [pipx](https://pipx.pypa.io/) for installing pre-commit

### 1. Fork and Clone

```bash
# Fork the repo on GitHub, then clone your fork
git clone https://github.com/<your-username>/garmin-coach-docs.git
cd garmin-coach-docs
```

### 2. Add the Upstream Remote

```bash
git remote add upstream https://github.com/askb/garmin-coach-docs.git
git fetch upstream
```

### 3. Keep Your Fork Up to Date

```bash
git checkout main
git pull upstream main
```

---

## Pre-commit Hook Setup

This repository uses [pre-commit](https://pre-commit.com/) to enforce
consistent formatting and catch common mistakes before every commit.

### Install pre-commit

```bash
# Using pip
pip install pre-commit

# Or using pipx (recommended to avoid polluting the system Python)
pipx install pre-commit
```

### Install the Hooks

Run this once from the root of the repository:

```bash
pre-commit install
```

### Run the Hooks Manually

To check all files without making a commit:

```bash
pre-commit run --all-files
```

### Hooks Enabled

| Hook | Purpose |
|------|---------|
| `trailing-whitespace` | Removes trailing whitespace |
| `end-of-file-fixer` | Ensures files end with a newline |
| `check-yaml` | Validates YAML syntax |
| `check-added-large-files` | Blocks files larger than 500 KB |
| `check-merge-conflict` | Detects leftover merge conflict markers |
| `detect-private-key` | Prevents accidental secret commits |
| `yamllint` | Lints YAML files against `.yamllint` |
| `markdownlint` | Lints Markdown files (MD013, MD033, MD041 disabled) |
| `check-github-workflows` | Validates GitHub Actions workflow files |
| `codespell` | Spell-checks source files |

---

## Documentation Style Guidelines

Follow these conventions to keep the documentation consistent and readable.

### Markdown Formatting

- Use a single `#` h1 heading per file (the document title).
- Use heading levels in order: `##` → `###` → `####` — never skip levels.
- Wrap lines at **120 characters** where possible.
- Use fenced code blocks with a language identifier:

  ````markdown
  ```bash
  echo "hello"
  ```
  ````

- Use `**bold**` for UI labels and important terms; use `_italics_` sparingly.
- Prefer ordered lists (`1.`) for sequential steps and unordered lists (`-`)
  for non-sequential items.

### Links and Cross-references

- Use relative links to reference other docs in this repository:

  ```markdown
  See the [Architecture](docs/architecture.md) document for details.
  ```

- Use absolute URLs for external resources.

### Commit Messages

All commits must follow this format (DCO and co-author trailers are required):

```text
type(scope): short description

Optional longer explanation.

Signed-off-by: Your Name <you@example.com>
Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

**Allowed types:** `fix`, `feat`, `chore`, `docs`, `style`, `refactor`

Use **lowercase** for the type and description.

### Spell-checking

Add any domain-specific terms (e.g., acronyms, proper nouns) that are not in
the standard dictionary to `.codespell-ignore`, one word per line.

---

## Submitting Changes via PR

1. **Create a feature branch** off `main`:

   ```bash
   git checkout -b docs/my-improvement
   ```

2. **Make your changes** and verify them with pre-commit:

   ```bash
   pre-commit run --all-files
   ```

3. **Commit** using the [commit format](#commit-messages) above:

   ```bash
   git commit -s -m "docs(scope): describe your change"
   ```

   The `-s` flag adds the required `Signed-off-by` trailer automatically.

4. **Push** to your fork:

   ```bash
   git push origin docs/my-improvement
   ```

5. **Open a Pull Request** against `askb/garmin-coach-docs`'s `main` branch
   on GitHub.

   - Write a clear PR title following the commit message format.
   - Describe *what* changed and *why* in the PR body.
   - Link the related issue with `Closes #<issue-number>` if applicable.

6. **Address review feedback** by pushing additional commits to the same branch.
   Avoid force-pushing after a review has started unless explicitly asked.

7. A maintainer will merge the PR once all checks pass and at least one
   approval is received.
