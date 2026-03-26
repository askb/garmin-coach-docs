<!--
SPDX-FileCopyrightText: 2026 Anil Belur <askb23@gmail.com>
SPDX-License-Identifier: Apache-2.0
-->

<!-- Sync Impact Report
Previous version: 0.0.0
Current version:  1.0.0
Impact level:     MAJOR — initial ratification
Summary:          First constitution for the GarminCoach documentation
                  repository. Establishes principles for documentation
                  quality, cross-repo accuracy, and governance.
-->

# GarminCoach Docs — Repository Constitution

**Version:** 1.0.0
**Ratified:** 2026-03-26
**Maintainer:** Anil Belur <askb23@gmail.com>

---

## Principles

### Principle I: Documentation Accuracy (NON-NEGOTIABLE)

- All documentation MUST reflect the current state of the codebase.
- When addon or app repos change, corresponding docs MUST be updated.
- Architecture diagrams, API references, and configuration examples must
  be validated against the actual implementation.
- Stale or outdated documentation is worse than no documentation.

### Principle II: No Secrets or PII (NON-NEGOTIABLE)

- Documentation MUST NOT contain real IP addresses, passwords, tokens,
  API keys, or personally identifiable information.
- Use placeholder values: `<YOUR_HA_IP>`, `<YOUR_TOKEN>`, `example.local`.
- Real infrastructure details belong in private repos, not public docs.

### Principle III: Atomic Commit Discipline (NON-NEGOTIABLE)

- Use **Conventional Commits** with lowercase types:
  `fix`, `feat`, `chore`, `docs`, `style`, `refactor`.
- Title: maximum 72 characters.
- Body: maximum 72 characters per line.
- Each commit represents exactly one logical change.

### Principle IV: Licensing & Attribution Standards (NON-NEGOTIABLE)

- Every source file MUST carry an SPDX header:

  ```text
  SPDX-License-Identifier: Apache-2.0
  SPDX-FileCopyrightText: YYYY Anil Belur <askb23@gmail.com>
  ```

- REUSE compliance is enforced by pre-commit hooks.

### Principle V: Agent Co-Authorship & DCO (NON-NEGOTIABLE)

- All agent-assisted commits MUST include:

  ```text
  Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
  ```

- All commits MUST include a DCO sign-off:

  ```text
  Signed-off-by: Anil Belur <askb23@gmail.com>
  ```

### Principle VI: Pre-Commit Integrity (NON-NEGOTIABLE)

- Active hooks: `markdownlint`, `yamllint`, `codespell`, `actionlint`.
- **Never** use `--no-verify` to bypass hooks.
- Run `pre-commit run --all-files` before every push.

### Principle VII: Cross-Repository Consistency

- This repo documents three codebases: addon, app, and ha-config.
- Architecture docs must reference the correct repo for each component.
- API endpoint docs must match tRPC router definitions in the app repo.
- Addon configuration docs must match `config.json` schema.

### Principle VIII: Spec-Driven Development

- Feature specifications live in `specs/NNN-feature-name/`.
- Each spec contains: `spec.md`, `plan.md`, `tasks.md`.
- Specs are the single source of truth for feature requirements.
- Agents use specs to understand context and generate implementations.

---

## Development Standards

### Git Workflow

1. Create a feature branch from `main`.
2. Make changes following Principle III (atomic commits).
3. Run `pre-commit run --all-files` before pushing.
4. Open a pull request with a clear description.
5. Squash-merge after approval.

### Documentation Structure

```text
docs/                    # User-facing documentation
├── ha-addon-mvp.md     # MVP design document
├── api-reference.md    # API endpoint reference
└── ...
specs/                   # Feature specifications
├── NNN-feature-name/
│   ├── spec.md         # Requirements & scenarios
│   ├── plan.md         # Technical implementation plan
│   └── tasks.md        # Task breakdown
```

### Markdown Standards

- Use ATX-style headings (`#`, `##`, `###`).
- Code blocks must specify a language tag.
- Links to other repos use full GitHub URLs.
- Tables must be properly aligned.

---

## Governance

### Amendment Process

1. Propose an amendment by opening an issue titled
   `Constitution Amendment: <summary>`.
2. Apply the amendment via a pull request that updates this file.

### Versioning

- **MAJOR**: New NON-NEGOTIABLE principles or removal of existing ones.
- **MINOR**: New negotiable principles or standards.
- **PATCH**: Clarifications, typo fixes, or formatting.
