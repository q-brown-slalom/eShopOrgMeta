---
title: "<Feature name>"
repo: "<repo from registry | cross-repo>"
status: Draft            # Draft | Approved | Implemented | Superseded
size: Project            # Small change | Project
authors: []
created: YYYY-MM-DD
approved_by: []
implemented_in: []       # links to merged product-repo PRs
plan: "plans/<repo>/YYYY-MM-DD-<topic>.md"
adrs: []
---

# <Feature name> — Design

## Intent and why

<The problem, who has it, and why now. Link the issue.>

- Personas / journeys affected (architecture/personas-and-journeys.md):
- Business rules added/changed (IDs from architecture/business-rules-catalog.md):
- NFR tier (standards/nfrs-and-slos.md):

## Goals / Non-goals / Anti-goals

- Goals:
- Non-goals (not doing now):
- Anti-goals (must not happen):

## Constraints

<Standards that apply (link standards/*), deadlines, compatibility, regulatory.>

## Approaches considered

| Option | Trade-offs | Chosen? |
|--------|-----------|---------|
| A | | |
| B | | |

Why the chosen option:

## Design decisions

### Affected repos and services

| Repo | Service/project | Change |
|------|-----------------|--------|

### Contracts

- HTTP API (endpoints, version, request/response, errors):
- gRPC (proto changes — additive only):
- Integration events (new/changed, publisher, consumers):
- Data (entities, migrations, backfill, retention tier per standards/data-retention.md):

### Domain

<New terms (add to architecture/domain-glossary.md), invariants, state transitions.>

### Security

<Scopes/roles, data sensitivity, OWASP checklist items from standards/security.md.>

### Observability

<Logs, traces, metrics, health checks added.>

### Rollout

<Feature flag, migration order, backward compatibility, rollback.>

## Acceptance criteria

- [ ] Given … when … then …

## Test strategy

<Unit / functional / e2e coverage per standards/testing.md.>

## Left to the builder

<Decisions intentionally not made here.>

## Open questions

-
