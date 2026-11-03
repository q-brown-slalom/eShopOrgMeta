# 0003. Store all specs and plans in eShopOrgMeta

- Status: Accepted
- Date: 2026-11-02
- Deciders: Q Brown

## Context

Superpowers defaults to `docs/superpowers/specs` and `docs/superpowers/plans` in the repo being
changed. Several upcoming features will span repos (e.g. admin app editing rules consumed by
eShop services).

## Decision

All design specs and implementation plans live in eShopOrgMeta:
`specs/<repo>/YYYY-MM-DD-<topic>-design.md` and `plans/<repo>/YYYY-MM-DD-<topic>.md`, with
`<repo> = cross-repo` for multi-repo work. Product-repo PRs link to them.

## Consequences

- Positive: one searchable history of intent; cross-repo specs have a natural home; specs are
  reviewed independently of code.
- Negative: two PRs per feature (spec in Meta, code in product repo); links can rot if files move —
  never rename spec files after approval.
