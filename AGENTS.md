# eShop Org Meta — Agent Guide

This repo is the **organization-level memory** for all eShop repos. It holds ways of working,
engineering standards, architecture knowledge, decisions, templates, and **all specs and plans**.
Product repos (e.g. `eShop`) contain code; this repo contains the knowledge used to build it.

## Load order

When working on any eShop org task, read only what the task needs, in this order:

1. [repos/registry.md](repos/registry.md) — which repo owns what.
2. [architecture/](architecture/): system context, service catalog, glossary, integration events,
   [business rules](architecture/business-rules-catalog.md),
   [personas and journeys](architecture/personas-and-journeys.md),
   [environments](architecture/environments-and-deployment.md).
3. [standards/](standards/): the standard(s) relevant to the change, plus
   [NFRs/SLOs](standards/nfrs-and-slos.md), [data retention](standards/data-retention.md), and the
   [tech radar](standards/tech-radar.md) before adding any dependency.
4. [decisions/](decisions/) — accepted ADRs override standards when they conflict.
5. [lessons-learned/](lessons-learned/) — check for known pitfalls in the affected area.

Repo-local instructions (a product repo's own `AGENTS.md` / `.github/copilot-instructions.md`)
override org standards **only** where an ADR in that repo or here allows it.

## Spec-driven workflow (Superpowers)

Superpowers skills are vendored in [.github/skills/](.github/skills/). Before any build or
fix task, load the `using-superpowers` skill and follow it. Workflow details:
[ways-of-working/spec-driven-development.md](ways-of-working/spec-driven-development.md).

**Location override (org preference, takes precedence over Superpowers defaults):**

| Artifact | Path in this repo |
|----------|-------------------|
| Design spec | `specs/<repo>/YYYY-MM-DD-<topic>-design.md` |
| Implementation plan | `plans/<repo>/YYYY-MM-DD-<topic>.md` |

- `<repo>` is the registry name (e.g. `eShop`). Use `cross-repo` when more than one repo changes.
- Design specs follow [templates/design-spec.md](templates/design-spec.md).
- Never write specs or plans into a product repo's `docs/superpowers/` folder.

## Memory maintenance rules

- Facts must cite a source (file path, ADR, or PR). Mark unverified items `Status: Proposed`.
- Each memory file has front matter with `status`, `owner`, `last_reviewed`. Update
  `last_reviewed` when you verify a file is still accurate.
- A change to a standard that alters behavior requires an ADR in [decisions/](decisions/).
- After an incident or a costly mistake, add an entry to [lessons-learned/](lessons-learned/).
- Keep files short and scannable; link instead of duplicating.

## Layout

| Folder | Contents |
|--------|----------|
| `ways-of-working/` | Branching, PRs, reviews, DoR/DoD, spec workflow, onboarding |
| `standards/` | Coding, API, architecture patterns, testing, security, observability, CI/CD, NFRs/SLOs, tech radar |
| `architecture/` | System context, service catalog, glossary, integration events, business rules, personas/journeys, environments |
| `decisions/` | Org-level ADRs |
| `repos/` | Repo registry |
| `lessons-learned/` | Postmortems and lessons |
| `templates/` | Spec, ADR, PR, issue, postmortem, repo onboarding templates |
| `specs/`, `plans/` | Superpowers outputs for all repos |
| `.github/skills/` | Vendored Superpowers skills |
| `.github/instructions/`, `.github/prompts/` | Scoped Copilot instructions and reusable prompts |
