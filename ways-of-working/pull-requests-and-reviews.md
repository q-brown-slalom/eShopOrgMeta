---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
---

# Pull requests and reviews

## Authoring

- Use [templates/pull-request.md](../templates/pull-request.md).
- Link the spec and plan in eShopOrgMeta (required for anything beyond a small change).
- Keep PRs reviewable (aim < 400 changed lines excluding generated files/migrations).
- PR must be green: build, unit/functional tests, e2e (when UI touched), markdownlint.
- Self-review first with the `requesting-code-review` skill.

## Reviewing

- At least **1** approval from a CODEOWNER of the affected area; **2** for changes to
  `src/Identity.API`, integration event contracts, DB migrations, or `eShop.AppHost`.
- Review against the spec first (does it do what was agreed?), then the standards.
- Severity labels in comments: `blocking:`, `suggestion:`, `nit:`, `question:`.
- Authors respond to feedback using the `receiving-code-review` skill: verify, don't blindly accept.

## Reviewer checklist

- [ ] Matches the approved spec; no unrequested scope.
- [ ] Tests written first and meaningful ([standards/testing.md](../standards/testing.md)).
- [ ] Contracts (API, gRPC, integration events) backward compatible or versioned.
- [ ] Security ([standards/security.md](../standards/security.md)) and observability considered.
- [ ] No new warnings (`TreatWarningsAsErrors` is on).
