---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
---

# Definition of Ready and Done

## Ready (before implementation starts)

- [ ] Problem and intent stated; goals and non-goals listed.
- [ ] Affected repos/services identified against [repos/registry.md](../repos/registry.md).
- [ ] Design spec in `specs/<repo>/` is `Approved` (or small change agreed in the issue).
- [ ] Contract changes (REST, gRPC, integration events, DB schema) called out.
- [ ] Acceptance criteria are testable.
- [ ] Dependencies and open questions resolved or explicitly deferred.

## Done (before closing the work item)

- [ ] Implementation plan tasks complete; plan updated if it diverged.
- [ ] Tests: unit + functional for behavior; Playwright e2e for user-visible flows.
- [ ] CI green; no new warnings.
- [ ] OpenAPI passes Spectral rules; new endpoints have summaries/descriptions.
- [ ] Telemetry: meaningful logs/traces for new flows; health checks for new dependencies.
- [ ] Docs updated: [architecture/](../architecture/) (service catalog, events, glossary, business rules,
      personas/journeys) if changed.
- [ ] Spec front matter set to `Implemented` with link to the merged PR.
- [ ] Lessons captured in [lessons-learned/](../lessons-learned/) if anything surprised us.
