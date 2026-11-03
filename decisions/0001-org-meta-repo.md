# 0001. Use eShopOrgMeta as org-level memory

- Status: Accepted
- Date: 2026-11-02
- Deciders: Q Brown

## Context

The org will grow beyond `eShop` (additional offering web apps, a business-rules admin app).
Standards, architecture knowledge, and decisions duplicated per repo drift, and AI agents need one
authoritative, citable source of context when building specs.

## Decision

Create `eShopOrgMeta` as the single source of truth for ways of working, standards, architecture,
decisions, templates, lessons learned, and the repo registry. Content is Markdown written for both
humans and agents, entered via `AGENTS.md`.

## Consequences

- Positive: one place to update; consistent agent behavior across repos.
- Negative: must be kept current; stale memory misleads agents. Mitigated by `last_reviewed`
  front matter and DoD step to update architecture docs.
