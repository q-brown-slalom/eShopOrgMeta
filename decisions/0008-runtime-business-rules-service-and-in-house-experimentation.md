---
status: Proposed
owner: Q Brown
last_reviewed: 2026-11-03
---

# 0008. Runtime business rules service and in-house experimentation

- Status: Proposed
- Date: 2026-11-03
- Deciders: Q Brown
- Scope: org
- Related: [spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md),
  [plan](../plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments.md),
  [business-rules-catalog.md](../architecture/business-rules-catalog.md) gaps G-01 and G-03,
  [ADR-0005](0005-deploy-to-azure-container-apps.md), [ADR-0006](0006-tiered-order-retention.md)

## Context

The business wants to run pricing, promotion, and fulfilment experiments, including several A/B tests
at once on different pages. Two gaps in the
[business-rules catalog](../architecture/business-rules-catalog.md) block this:

- **G-01:** no pricing rules exist. Nothing sets promotions or shipping fees, and the storefront, not
  the server, decides the price (Ordering trusts the client's `UnitPrice`).
- **G-03:** all rules are deploy-time config or code. There is no runtime rule store, audit trail, or
  effective-dating, so every change needs a redeploy and can't be audited.

Ordering.API, WebApp, and mobile-bff are T1 ([nfrs-and-slos.md](../standards/nfrs-and-slos.md)).
Whatever owns the rules must not become a runtime dependency of checkout. The user chose an in-house
engine over a third-party experimentation product.

## Options considered

1. **New `BusinessRules.API` service** that owns rule definitions, versions, audit, experiments,
   assignment config, tracking ingest, and results. Consumers evaluate rules locally from a cached,
   event-refreshed snapshot. Pros: one admin API, one audit trail, one owner for cross-page
   experiments; matches service-owned data and event patterns. Cons: one more (T3) service to operate;
   consumers need a fallback when it's down.
2. **Each service owns its own rules and admin endpoints**, with a shared assignment library. Pros: no
   new service. Cons: audit, effective-dating, and admin built four times; eShopAdmin talks to four
   APIs; cross-page experiments (shown on the item page, charged at checkout) have no owner.
3. **Microsoft.FeatureManagement variants + Azure App Configuration.** Pros: off-the-shelf flags and
   variants. Cons: effectively a third-party route; not on the [tech radar](../standards/tech-radar.md);
   config-shaped, a poor fit for prices and business-user audit; no local-dev story without Azure.

A generic rules engine or DSL editable by business users was dropped early: it violates the spec's
anti-goal (rule *types* are code, rule *values* are data) and is hard to validate and test.

## Decision

Option 1. Build a new **BusinessRules.API** (T3, Postgres `rulesdb`, deployed to ACA via Aspire per
ADR-0005) and a shared **eShop.BusinessRules.Client** library. Consumers load the published snapshot
at startup, refresh it on `BusinessRulesSnapshotPublishedIntegrationEvent`, and poll every 60 s.
If no snapshot is available they use built-in defaults equal to today's behaviour.

Two mechanisms the spec left to the builder are decided here:

- **Service-to-service auth = OAuth client credentials via Identity.API** (plan decision D2). Clients:
  `businessrules-reader` (scope `businessrules`, used by Ordering.API to read the snapshot) and
  `businessrules-api` (scope `ordering.usage`). Secrets are Aspire `AddParameter(..., secret: true)`.
  Tokens come from a small `ServiceTokenHandler` (plain `HttpClient` POST to `/connect/token`, no new
  package). Identity.API stays the only token issuer.
- **Publish guard data source** (plan decision D3). Ordering.API counts v1 create-order calls per UTC
  day in a new table `ordering.apiusagedaily` and exposes `GET api/usage/create-order-v1?days=7`
  (scope `ordering.usage`). BusinessRules.API reads it before publishing a price-affecting rule and
  **fails closed** (503) if it can't. BusinessRules.API can't read OTel metrics and cross-database
  reads are banned; the OTel counter `ordering.create_order.v1.calls` is still emitted for dashboards
  and alerts.

## Consequences

- Positive: business admins change, schedule, and audit rules without a deployment; one owner per
  experiment; the server computes every price the shopper pays.
- Negative: a new T3 service, database, and two service clients to operate and secure; Ordering.API
  gains a usage table and endpoint for the guard; publishing price rules is blocked while v1 traffic
  exists or the usage endpoint is unreachable.
- **T1 independence rule:** Ordering.API, WebApp, and mobile-bff never `WaitFor` BusinessRules.API in
  the AppHost and never call it synchronously while pricing or creating orders. Checkout keeps working
  on the last snapshot, or on defaults, when BusinessRules.API is down.
- Follow-ups (standards/docs to update):
  - Phase 1 (this plan): service catalog, system context, integration events, domain glossary,
    environments (`rulesdb`), data retention (new entities and tiers).
  - Phase 2: business-rules catalog entries PRC-001, PRM-001, SHP-001 and runtime mechanisms for
    ORD-001 and ORD-013.
  - Phase 3: onboard eShopAdmin (repo registry, onboarding checklist); journey J8 becomes real.
  - Phase 4: experiments — `OrderPricedIntegrationEvent`, new journey J10 *Run an A/B test*.
  - Phase 5: tracking and results — tracking consent and retention sign-off, environments (Key Vault
    HMAC secret).
