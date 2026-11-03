---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
---

# Branching and commits

## Branching

- Trunk-based: `main` is always releasable and protected (PR + passing checks required).
- Short-lived branches, merged within a few days:
  - `feature/<short-topic>`
  - `fix/<short-topic>`
  - `chore/<short-topic>`
  - `docs/<short-topic>`
- Superpowers `using-git-worktrees` creates the branch; keep the same `<topic>` as the spec file.
- Squash-merge into `main`; delete the branch after merge.

## Commits

- [Conventional Commits](https://www.conventionalcommits.org/): `type(scope): summary`
  - Types: `feat`, `fix`, `refactor`, `test`, `docs`, `build`, `ci`, `chore`, `perf`
  - Scope = service/project, e.g. `feat(ordering): add order notes`
- Small, frequent commits during TDD (red → green → refactor) are expected; the squash message
  is what lands on `main`.
- Never commit secrets, connection strings, or real customer data.
