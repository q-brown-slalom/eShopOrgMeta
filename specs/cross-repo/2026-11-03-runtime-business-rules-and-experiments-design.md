---
title: "Runtime business rules and A/B experiments"
repo: "cross-repo"
status: Approved         # Draft | Approved | Implemented | Superseded
size: Project            # Small change | Project
authors: ["Q Brown", "Copilot (brainstorming)"]
created: 2026-11-03
approved_by: ["Q Brown (2026-11-03)"]
implemented_in: []       # links to merged product-repo PRs
plan: "plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments.md"
adrs: ["ADR-0008 (to be written): runtime business rules service and in-house experimentation"]
---

# Runtime business rules and A/B experiments — Design

## Intent and why

The business wants to run experiments on **pricing, promotions, and fulfilment strategies**, including
**several A/B tests at the same time on different pages**, and judge them on **conversion, order
revenue, page drop-off, and clicks**. Today every rule is deploy-time config or code
([business-rules-catalog.md](../../architecture/business-rules-catalog.md), gap **G-03**), there are no
pricing rules at all (**G-01**), and the storefront — not the server — decides the price
(see *Current state*). Any experiment therefore needs a full redeploy and can't be enforced or measured.

**This work builds the tools, not the strategies.** Engineering delivers: a runtime rule store,
server-side pricing, an A/B experiment engine, behavioural tracking, results reporting, and the
fulfilment capabilities strategies need (partial fulfilment, per-order grace period). Choosing prices,
promotions, thresholds, and which tests to run belongs to the business (persona P5).

- Personas / journeys affected ([personas-and-journeys.md](../../architecture/personas-and-journeys.md)):
  P5 Business admin (primary — J8 *Edit a business rule* becomes real; new journey *Run an A/B test*);
  P1/P2/P3 shoppers (J1 browse, J2 cart, J3 checkout, J4 track order, J6 mobile purchase see variant
  prices and are tracked); P6 operator (new service and stale-rules alerts).
- Business rules added/changed ([business-rules-catalog.md](../../architecture/business-rules-catalog.md)):
  new **PRC-001** price adjustment, **PRM-001** promotion, **SHP-001** shipping fee; changed **ORD-001**
  grace period (config → runtime, per order), **ORD-013** stock shortfall policy (code → runtime, adds
  `PartialFulfilment`). Addresses gaps **G-01** (pricing rules) and **G-03** (runtime store, audit,
  effective-dating).
- NFR tier ([nfrs-and-slos.md](../../standards/nfrs-and-slos.md)): new **BusinessRules.API = T3**
  (internal/admin); Ordering.API, WebApp, mobile-bff stay **T1** and must not take a runtime dependency on
  a T3 service (see *Resilience*). eShopAdmin = T3.

### Current state (verified in code, 2026-11-03)

| Fact | Source |
|------|--------|
| Basket stores only `product_id` + `quantity` | `eShop/src/Basket.API/Proto/basket.proto` |
| WebApp reads `CatalogItem.Price` and sends `UnitPrice` (and `OldUnitPrice`) in `CreateOrderRequest`; Ordering trusts the unit price. No discount is sent: `BasketItemExtensions.ToOrderItemDTO` never sets `Discount`, so it is always 0 | `eShop/src/WebApp/Services/BasketState.cs`, `Ordering.API/Extensions/BasketItemExtensions.cs`, `Ordering.API/Application/Commands/CreateOrderCommandHandler.cs` |
| `Order.GetTotal()` = Σ units × unit price — ignores `Discount` (harmless today because discount is always 0; wrong once promotions exist) | `Ordering.Domain/AggregatesModel/OrderAggregate/Order.cs` |
| Ordering.API has no Catalog client today (new dependency); Catalog exposes a batch read `GET api/catalog/items/by?ids=` | `Catalog.API/Apis/CatalogApi.cs` |
| Any stock-rejected item cancels the whole order (ORD-013) | `Order.SetCancelledStatusWhenStockIsRejected` |
| Stock is checked (not reserved) before payment; decremented on `Paid` | [integration-events.md](../../architecture/integration-events.md), CAT-004 |
| Grace period is one global config value read by OrderProcessor (ORD-001) | `OrderProcessor/Services/GracePeriodManagerService.cs`, `appsettings.json` |
| No analytics/behavioural data exists; only OTel (logs 30 d, traces 7 d) | [data-retention.md](../../standards/data-retention.md) |

## Goals / Non-goals / Anti-goals

- **Goals**
  1. Business admins change rule values and schedule them (effective-dated) **without a deployment**,
     with a full audit trail.
  2. Business admins run **multiple concurrent A/B experiments**, each overriding one or more rules, on
     different pages, with sticky assignment.
  3. The **server** computes every price the shopper pays; displayed price = charged price.
  4. Each order records its experiment assignments and pricing breakdown, so results are attributable.
  5. Results per variant: **conversion, order revenue, page drop-off, clicks**, plus fulfilment
     metrics (**order-to-ship time, cancellation rate, partial-fulfilment rate**).
  6. Capabilities needed by fulfilment strategies exist: **partial fulfilment** as a selectable ORD-013
     policy, and a **per-order, runtime-editable grace period** (ORD-001).
- **Non-goals (not now)**
  - Choosing or implementing any actual price, promotion, threshold, or experiment design.
  - Tax calculation, multi-currency, coupon codes, loyalty, personalised pricing.
  - Carrier selection, warehouses, delivery-speed tiers, split shipments over time (partial fulfilment
    ships what is in stock once and drops the rest).
  - Multi-armed bandits, sequential testing, automatic winner promotion, a third-party experimentation
    or analytics product.
  - Per-offering (eShopOffering) rule scoping — design keeps a `scope` field for it; only the default
    scope is used now.
- **Anti-goals (must not happen)**
  - A free-form rules language or scripts editable by business users. Rule *types* are code; rule
    *values* are data.
  - Client-supplied prices, discounts, or variant choices being trusted by the server.
  - Checkout failing or slowing beyond its SLO because BusinessRules.API (T3) is down.
  - A shopper seeing one price and being charged another without being told first.
  - Domain invariants (ORD-003, ORD-005…ORD-012, CAT-003, IDN-*, WHK-002, RET-*) becoming editable.
  - Personal data (names, emails, addresses, free text) in tracking events.
  - Changing the definition of a running experiment in a way that invalidates its results.

## Constraints

- Org standards: [architecture-patterns.md](../../standards/architecture-patterns.md) (service-owned data,
  outbox, idempotent handlers, DDD where rules are rich), [api-design.md](../../standards/api-design.md)
  (Minimal APIs, versioning, ProblemDetails, Spectral), [security.md](../../standards/security.md) (admin
  scope + role, no PII in logs), [observability-and-resilience.md](../../standards/observability-and-resilience.md),
  [testing.md](../../standards/testing.md) (TDD, MSTest + NSubstitute, Playwright),
  [dotnet-coding.md](../../standards/dotnet-coding.md), [data-retention.md](../../standards/data-retention.md)
  (every new entity states its tier).
- [Tech radar](../../standards/tech-radar.md): use only **Adopt** items (Postgres/EF Core, RabbitMQ +
  outbox, Minimal APIs, Blazor + WebAppComponents, YARP, Aspire). No new third-party dependency.
- [ADR-0005](../../decisions/0005-deploy-to-azure-container-apps.md): new service deploys to ACA via Aspire.
- [ADR-0006](../../decisions/0006-tiered-order-retention.md): new order fields follow the order tiers.
- [ADR-0007](../../decisions/0007-tokenize-card-data.md): the future Stripe charge must use the
  server-computed (and possibly reduced, after partial fulfilment) order total.
- Jurisdictions US + Canada: behavioural tracking of shoppers is subject to CCPA/CPRA, PIPEDA and
  Quebec Law 25 (see *Open questions* on consent).
- eShopAdmin is a **planned** repo: it must be onboarded with
  [repo-onboarding-checklist.md](../../templates/repo-onboarding-checklist.md) before its phase starts.

## Approaches considered

| Option | Trade-offs | Chosen? |
|--------|-----------|---------|
| **A. New `BusinessRules.API` service** owning rule definitions, versions, audit, experiments, assignment config, tracking ingest and results; consumers evaluate from a locally cached, event-refreshed snapshot | + One admin API, one audit trail, one owner for cross-page experiments; matches service-owned-data and event patterns. − One more service (T3) to operate; consumers need a fallback when it's down | **Yes** |
| B. Each service owns its own rules + admin endpoints; shared assignment library | + No new service. − Audit/effective-dating/admin built 4×; eShopAdmin talks to 4 APIs; cross-page experiments (shown on item page, charged at checkout) have no owner | No |
| C. Microsoft.FeatureManagement variants + Azure App Configuration | + Off-the-shelf flags/variants. − Effectively a third-party route (user chose in-house); not on tech radar (ADR); config-shaped, poor fit for prices and business-user audit; no local-dev story without Azure | No |
| (dropped early) Generic rules engine / DSL editable by business | Violates anti-goal; hard to validate and test | No |

**Why A:** the user chose an in-house engine and option A. It gives the business one place to work,
one history, and a single owner per experiment, while the T1 checkout path stays independent by
evaluating rules locally from a cached snapshot.

## Design decisions

Items marked **(call)** were decided while writing this spec, not by the requester; reviewers should
check them.

### Core model

- **Rule type** (code): a named, versioned contract with an ID from the catalog, a typed parameter
  schema, validation ranges, a **built-in default equal to today's behaviour**, and the pages/services
  that read it. Adding a rule type requires a release; changing its value does not.
- **Rule version** (data): a value for a rule type in a scope, with `EffectiveFrom` (optional
  `EffectiveTo`), status `Draft → Published → Superseded`, author, publisher, and timestamps.
  Published versions are immutable.
- **Rule entries (targeted rule types) (call):** PRC-001 and PRM-001 line promotions are *targeted*.
  A rule version for them is one document per scope holding a **list of entries**, each with a target
  (catalog item / catalog type / catalog brand / all). When several entries match a line, the **most
  specific target wins**: item > type > brand > all. PRM-001 order promotions, SHP-001, ORD-001 and
  ORD-013 are *global* (one value per scope).
- **Rules snapshot**: an immutable, monotonically versioned document containing all published rule
  versions (including future-dated ones) and all `Running` experiments. Consumers evaluate rules
  against the snapshot plus "now" — effective-dating needs no extra publish.
- **Experiment**: `Key`, name, hypothesis, primary metric, **audience** (see *Assignment*), traffic
  allocation (% of eligible subjects enrolled), **pages** where exposure is recorded, start/end time,
  and 2–4 **variants**. Exactly one variant is `control` with no overrides; each other variant holds
  **rule overrides** (validated against the rule type schema). Weights sum to 100.
  Status: `Draft → Scheduled → Running → Stopped | Completed`.
  - **Override lock (call):** the lock unit is the *rule + target*, not the whole rule type, so several
    pricing/promotion tests can run at once.
    - Global rules (order promotion, SHP-001, ORD-001, ORD-013): at most one `Running`/`Scheduled`
      experiment each.
    - Targeted rules (PRC-001, PRM-001 line): experiments on the same rule type must use the **same
      target kind** and **disjoint target ids**; a target of `all` is exclusive for that rule type.
      This avoids needing catalog membership data (item ∈ brand) inside BusinessRules.API.
    - An override entry replaces the published entry for the same target and otherwise adds to it;
      most-specific-wins then applies as usual.
    - Different rules run concurrently with independent assignment (different hash salt).
  - **(call)** At most **10** concurrently running experiments.
  - **(call)** Once `Running`, variants, weights, audience, and traffic allocation are immutable; the
    only actions are *Stop* (everyone reverts to the published rules) and *Complete*. To change a
    test, stop it and create a new one.
  - **Promote variant**: copies a variant's overrides into a new **Draft** rule version; publishing it
    is a separate, audited action. No automatic promotion.

### Rule types in scope

| ID | Rule type | Parameters (validation) | Default (= today) | Evaluated in |
|----|-----------|-------------------------|-------------------|--------------|
| PRC-001 | Price adjustment | Target: catalog item / catalog type / catalog brand / all; mode: percent off (0–90%) or fixed unit price (> 0) | none → catalog price | Ordering.API pricing |
| PRM-001 | Promotion | Line promotion (target as PRC-001, percent or amount off, optional min units) or order promotion (amount/percent off when subtotal ≥ X); display label (≤ 60 chars, no HTML) | none | Ordering.API pricing |
| SHP-001 | Shipping fee | Flat fee (0–100), free-shipping threshold (optional, ≥ 0) | fee 0 | Ordering.API pricing |
| ORD-001 | Grace period | 1–60 minutes **(call)** | 1 minute (current config) | Ordering.API at order creation |
| ORD-013 | Stock shortfall policy | `AllOrNothing` \| `PartialFulfilment` | `AllOrNothing` | Ordering.API at creation |

ORD-002 (polling interval) stays config. All other catalog rules stay as they are.

### Pricing (server-side)

- A **pricing calculator** (pure, deterministic domain service) in a new shared library
  `eShop.BusinessRules.Client` takes catalog base prices + snapshot + assignments + basket lines and
  returns a **price quote**: per line base price, adjusted unit price, line discount, promotion labels;
  order subtotal, order discount, shipping fee, total, **snapshot version**, assignments used.
- Order of evaluation: PRC-001 adjusts unit price → best single line promotion per line (matches ORD-009
  "higher discount wins") → best single order promotion → SHP-001 on the discounted subtotal.
  No stacking of promotions of the same level **(call)**.
- **Ordering.API is the pricing authority.** It fetches base prices from Catalog.API (existing batch
  item endpoint, via service discovery + standard resilience handler) and evaluates locally.
- Order-level discount is allocated across lines proportionally to line totals so that
  `OrderItem.Discount` keeps ORD-010 (≤ line total) **(call)**.
- Fix `Order.GetTotal()` to `Σ(units × unitPrice − discount) + shippingFee`.
- Prices are rounded to 2 decimals, half away from zero, per line **(call)**.

### Assignment

- Deterministic bucketing: `bucket = hash(experimentKey + salt + subjectKey) mod 10 000`; enrolment
  if bucket < allocation; variant by weight ranges. Same inputs → same variant on every service and
  device; no assignment table is needed. Hash: a stable non-cryptographic 64-bit hash implemented in
  the shared library **(left to builder: exact algorithm, must be identical across .NET and stable
  across versions)**.
- **Subject key and audience (decided: include anonymous shoppers):** every experiment, including
  price, promotion, shipping and fulfilment experiments, covers anonymous and signed-in shoppers.
  - **Experiment subject id:** each identity user gets a persistent `ExperimentSubjectId` (GUID) on
    `ApplicationUser`, issued to clients as an access-token claim (`experiment_subject`). Server-side
    services read it from the validated token, so it cannot be tampered with.
  - **Anonymous visitors:** WebApp issues a **server-generated, data-protection-signed, HttpOnly**
    visitor cookie (random GUID, 1-year sliding expiry) **(call)**. Clients can't choose its value, only
    discard it. ClientApp requires sign-in before browsing (J6), so it always uses the claim.
  - **Linking at sign-in (call):** when a user signs in and their `ExperimentSubjectId` is empty, it is
    set to the current visitor id ("first device wins"). WebApp then re-keys the visitor cookie to
    the user's `ExperimentSubjectId`, so the shopper keeps the same variants on that device and gets
    them on every device after sign-in. A shopper browsing anonymously on a *second* device may see a
    different variant until they sign in. The price-consistency check at checkout catches the switch
    (see below). How the visitor id reaches Identity.API during sign-in is **left to the builder**.
  - **Anti-tampering:** checkout requires sign-in, so the **charged** price always comes from the
    token-bound `ExperimentSubjectId`. Clearing cookies only re-rolls the *displayed* anonymous
    variant. The 409 price-changed flow then shows the real price before purchase. Residual risk
    accepted: an anonymous shopper can re-roll what they see, and this is visible in results as a
    higher anonymous-subject count.
  - Experiments can still be restricted to audience `SignedIn` when the business wants it.
  - **Analysis link:** at sign-in WebApp/ClientApp send an `identify` tracking event that pairs the
    previous visitor id with the subject HMAC. Results use it to merge the shopper's pre-sign-in page
    views and clicks into their funnel. It's used only for analysis, never for assignment.
- **Exposure**: recorded the first time per page view that a subject renders a page listed in the
  experiment's `pages` while enrolled. Assignment is evaluated everywhere; *exposure* gates analysis
  (subjects never exposed are excluded).
- **Tracking consent (decided: consent limits tracking only):** WebApp and ClientApp keep a
  consent flag; the consent UI is **left to the builder**. Without consent, shoppers are **still
  enrolled** and see their variant's prices, but no tracking events (page view, click, exposure,
  identify) are sent. `OrderPricedIntegrationEvent` carries `TrackingConsent = false`, and
  BusinessRules.API doesn't store outcomes for those orders **(call)**. Non-consenting shoppers are
  therefore left out of results entirely, which keeps conversion denominators consistent. Legal
  sign-off on the consent wording and default is still needed (owner Q Brown).

### Price consistency at checkout

- New **Ordering.API `POST /api/pricing/quote`** returns a price quote for given product ids/units for
  the calling shopper (signed-in: `experiment_subject` claim; anonymous: visitor id forwarded by
  WebApp's server). WebApp cart, checkout, item and catalog pages, and ClientApp
  use it instead of `CatalogItem.Price`.
- New **create order v2**: request carries items (product id, units) and `expectedTotal` from the
  last quote; client price/discount fields are not accepted. If the server-computed total ≠
  `expectedTotal`, return **409 ProblemDetails** with the fresh quote; WebApp/ClientApp show "prices
  changed" and require reconfirmation.
- Create order **v1** stays for compatibility: it **ignores** client `UnitPrice` and charges server
  prices (no 409). Because v1 clients don't show quoted prices, **ClientApp moves to quote + v2 in
  phase 1 (call)**, and BusinessRules.API **refuses to publish price-affecting rules (PRC-001, PRM-001,
  SHP-001) while the `ordering.create_order.v1.calls` metric is non-zero over the last 7 days** unless a
  publisher explicitly overrides the guard with an audited reason **(call)**. v1 is removed in a
  follow-up once its traffic is zero.

### Fulfilment capabilities

- **Per-order grace period (ORD-001):** at creation Ordering resolves ORD-001 for the buyer (including
  experiment overrides) and stores `Order.GracePeriodEndsAt`. OrderProcessor's query uses that column
  instead of the global setting (keeps `CheckUpdateTime`, ORD-002). Existing rows are backfilled with
  `OrderDate + 1 minute`.
- **Stock shortfall policy (ORD-013):** resolved at creation and stored on the order
  (`StockShortfallPolicy`), so a later rule change doesn't affect in-flight orders. On
  `OrderStockRejectedIntegrationEvent`:
  - `AllOrNothing` → current behaviour (cancel).
  - `PartialFulfilment` and ≥ 1 line in stock → rejected lines are marked **unfulfillable** (kept on
    the order for history, excluded from totals), order is **repriced** (line prices and promotions as
    quoted; order-level discount re-allocated only across remaining lines and capped by ORD-010;
    shipping fee unchanged **(call)**), `Description` lists dropped items, order moves to
    `StockConfirmed`. No new `OrderStatus` value.
  - `PartialFulfilment` and 0 lines in stock → cancel (as today).
  - `OrderStatusChangedToPaidIntegrationEvent.OrderStockItems` excludes unfulfillable lines so Catalog
    (CAT-004) decrements only shipped items. Payment happens after stock confirmation, so the shopper is
    only charged the reduced total; no refunds are needed.
- WebApp `/user/orders` and ClientApp order detail show dropped lines and the adjusted total.

### Affected repos and services

| Repo | Service/project | Change |
|------|-----------------|--------|
| eShop | **BusinessRules.API** (new) | Rule types registry, rule versions, publish, audit, experiments, snapshot endpoint, tracking ingest, results aggregation, admin API. Postgres `rulesdb`. Publishes/consumes events |
| eShop | **eShop.BusinessRules.Client** (new shared lib) | Snapshot cache + refresh, built-in defaults, rule resolution, assignment hashing, pricing calculator. Used by Ordering.API and WebApp |
| eShop | Ordering.Domain / Infrastructure | Order pricing fields, experiment assignments, `GracePeriodEndsAt`, `StockShortfallPolicy`, unfulfillable lines, partial-fulfilment behaviour, `GetTotal` fix; EF migrations + backfill |
| eShop | Ordering.API | Pricing quote endpoint, create order v2 (+ v1 server-pricing), partial-fulfilment handling, new events |
| eShop | OrderProcessor | Use `GracePeriodEndsAt` |
| eShop | WebApp + WebAppComponents | Quote-based prices, v2 checkout with 409 handling, visitor id cookie, exposure + page-view + click tracking (`data-track` attribute), partial-fulfilment display, consume snapshot-published event |
| eShop | ClientApp (MAUI) | Quote-based prices, v2 checkout, tracking, partial-fulfilment display |
| eShop | mobile-bff | Routes for pricing quote and tracking ingest |
| eShop | Identity.API | `businessrules` / `businessrules.admin` scopes, eShopAdmin client (PKCE), roles, service client for snapshot reads; `ApplicationUser.ExperimentSubjectId` + `experiment_subject` claim, set from the visitor id at first sign-in |
| eShop | eShop.AppHost | `rulesdb`, BusinessRules.API resource and references |
| eShopAdmin (planned, new repo) | Blazor admin app | Rules list/edit/schedule/publish/history; experiments create/schedule/start/stop/complete/promote; results dashboards |
| eShopOrgMeta | architecture docs | Catalog, glossary, events, service catalog, system context, personas/journeys (see *Rollout* → docs) |

### Contracts

- **HTTP API — BusinessRules.API v1** (`api/rules`, `api/experiments`, `api/tracking`):
  - Admin (scope `businessrules.admin` + role): list rule types (schema, defaults, ranges); CRUD draft
    rule versions; publish; version history and audit log; experiment CRUD (draft), schedule, start,
    stop, complete, promote variant; results per experiment (see *Results*). Validation errors as
    400 ProblemDetails; conflicting experiment overrides as 409.
  - Runtime (internal only, not routed via mobile-bff; service credential with `businessrules` scope):
    `GET api/rules/snapshot` returns the latest snapshot; supports `If-None-Match` on snapshot version
    (304).
  - Tracking ingest: `POST api/tracking/events` — batch (≤ 50 events, ≤ 32 KB). Called by WebApp's
    server and by ClientApp via mobile-bff. Anonymous allowed, rate-limited per client IP and visitor
    id; strict schema (see *Data*). 202 Accepted.
- **HTTP API — Ordering.API:** `POST api/pricing/quote` (v1; anonymous or `orders` scope),
  `POST api/orders` **v2** (`expectedTotal`, 409 on mismatch), v1 behaviour change described above.
  Order read models gain pricing breakdown and unfulfillable flag (additive).
- **gRPC:** none.
- **Integration events** (add to [integration-events.md](../../architecture/integration-events.md)):

  | Event | Publisher | Consumers | Payload |
  |-------|-----------|-----------|---------|
  | `BusinessRulesSnapshotPublishedIntegrationEvent` | BusinessRules.API | Ordering.API, WebApp | `SnapshotVersion`, `PublishedAt` (consumers fetch the snapshot; event carries no rule data) |
  | `OrderPricedIntegrationEvent` | Ordering.API | BusinessRules.API | `OrderId`, subject key hash, assignments `[{experimentKey, variantKey}]`, subtotal, discount, shipping fee, total, snapshot version, `TrackingConsent`, `Reason` (`Created` \| `PartialFulfilment`) |

  BusinessRules.API also **consumes** existing `OrderStatusChangedToPaidIntegrationEvent`,
  `OrderStatusChangedToShippedIntegrationEvent`, `OrderStatusChangedToCancelledIntegrationEvent`
  (by `OrderId`). Changed (additive behaviour): `OrderStatusChangedToPaidIntegrationEvent` no longer
  lists unfulfillable lines. All publishing via outbox; handlers idempotent.
- **Data** (retention tier per [data-retention.md](../../standards/data-retention.md)):

  | Entity | Store | Notes | Retention |
  |--------|-------|-------|-----------|
  | `RuleVersion`, `Experiment`, `Variant`, `RuleOverride` | `rulesdb` | No personal data | Kept while referenced; superseded/completed kept **7 years** (pricing history supports financial audit) **(call)** |
  | `RulesSnapshot` | `rulesdb` | Serialized published state | Latest + all referenced by orders' snapshot versions for 7 years **(call)** |
  | `AuditEntry` | `rulesdb` | Who (admin user id), action, before/after, when | **7 years** **(call)** |
  | `TrackingEvent` | `rulesdb` (partitioned by day) | `type` (`page_view` \| `click` \| `exposure` \| `identify`), page key, element key (allow-listed pattern `^[a-z0-9-]{1,40}$`), subject key **HMAC** (secret key in Key Vault), visitor/session ids (random), experiment+variant keys, client (`web` \| `mobile`), timestamp. No URLs with query strings, no free text | **Raw: 90 days** **(call)**; erasure deletes rows by subject HMAC |
  | `OrderOutcome` | `rulesdb` | From events: order id, subject HMAC, assignments, totals, status timestamps | Same tiers as orders (ADR-0006); pseudonymous already |
  | `ExperimentResultSnapshot` | `rulesdb` | Aggregates only | **7 years** **(call)** |
  | `ApplicationUser.ExperimentSubjectId` | `identitydb` | Random GUID, pseudonymous | Lifetime of the identity account; deleted with it on erasure |
  | Visitor cookie | Browser | Signed random GUID | 1-year sliding expiry; set only after consent if the opt-in default applies to it (legal sign-off) |
  | Order: `Subtotal`, `DiscountTotal`, `ShippingFee`, `Total`, `PricingSnapshotVersion`, `GracePeriodEndsAt`, `StockShortfallPolicy`; `OrderItem.IsUnfulfillable`, `BasePrice`, `PromotionLabel`; `OrderExperimentAssignment` (order id, experiment key, variant key) | `orderingdb` | Financial / non-personal | Order tiers (ADR-0006); archived tier keeps them (financial fields) |

  Migrations are additive with backfill (`Total` from existing lines, `ShippingFee = 0`,
  `GracePeriodEndsAt = OrderDate + 1 min`, `StockShortfallPolicy = AllOrNothing`).

### Results (reporting)

- Aggregation job in BusinessRules.API runs every **15 minutes** **(call)**; dashboards in eShopAdmin
  show per variant, since experiment start:
  - **Exposed subjects.**
  - **Conversion** = exposed subjects with ≥ 1 order that reached `Paid` after first exposure ÷ exposed.
  - **Order revenue**: total and per exposed subject; average order value (server totals after
    partial fulfilment).
  - **Page drop-off**: funnel catalog → item → cart → checkout → order placed; share of sessions
    (30-minute inactivity) that reached step N but not N+1. Mobile views map to the same page keys.
  - **Clicks**: count and click-through rate per tracked element per page view.
  - **Fulfilment**: median and p90 order-to-ship time (`Submitted` → `Shipped`), cancellation rate,
    partial-fulfilment rate.
  - Difference vs control with **95% confidence interval** (two-proportion z-test for rates, Welch's
    t-test for revenue) and a "not enough data" label below **100 exposed subjects per variant**
    **(call)**. No sequential-testing corrections (non-goal); dashboard states this.

### Resilience

- Consumers load the snapshot at startup and on `BusinessRulesSnapshotPublishedIntegrationEvent`, and
  poll every **60 s** as a fallback **(call)**. If BusinessRules.API is unreachable they keep the last
  snapshot; on a cold start with no snapshot they use **built-in defaults (= today's behaviour)** and
  run every experiment as control.
- Publish → visible in all consumers: p95 ≤ 60 s (typically ≤ 5 s via event).
- Ordering.API pricing and order creation never call BusinessRules.API synchronously.
- Tracking ingest failures are dropped after one retry from WebApp (best-effort; never blocks page
  rendering). ClientApp buffers up to 200 events in memory.

### Domain

New glossary terms (added to [domain-glossary.md](../../architecture/domain-glossary.md) as Proposed):
rule type, rule version, rules snapshot, price quote, promotion, shipping fee, stock shortfall policy,
partial fulfilment, unfulfillable order item, experiment, variant (control / treatment), experiment
subject, assignment, exposure, tracking event.

Invariants:

- A published rule version is immutable; values always satisfy the rule type's schema and ranges.
- A global rule, or a targeted rule's target, is overridden by at most one running or scheduled
  experiment (see *Override lock*).
- A running experiment's design is immutable; exactly one control variant; weights sum to 100.
- Final unit price ≥ 0.01; `OrderItem.Discount` ≤ line total (ORD-010); order total ≥ 0.
- An order's pricing, assignments, grace period and shortfall policy are fixed at creation; only
  partial fulfilment may reprice it, and only downward.
- Unfulfillable lines can't become fulfillable again.

### Security

- eShopAdmin: Authorization Code + PKCE client; scope `businessrules.admin`.
  Roles **(call)**: `RulesEditor` (drafts, view results) and `RulesPublisher` (publish, schedule,
  start/stop/complete experiments, promote). Policies enforced server-side in BusinessRules.API.
  **No second-person approval** for publishing (decided); a publisher may publish their own drafts.
  Customer scopes are never reused ([security.md](../../standards/security.md)).
- Snapshot endpoint is internal-only (not exposed through mobile-bff or public ingress) and requires a
  service credential — unreleased promotions are commercially sensitive. Mechanism (client
  credentials vs. internal ingress + managed identity) **left to builder** within security.md.
- Server never trusts client prices, discounts, or variants; v2 create order has no price fields.
- Tracking: schema-validated, size-limited, rate-limited, allow-listed keys; no PII; subject key
  stored only as HMAC; IP addresses not stored.
- Every publish, schedule, start, stop, complete, promote is written to `AuditEntry` and logged as a
  security-relevant event (no PII).
- OWASP checklist items: access control (admin roles), input validation (FluentValidation on all
  commands), no sensitive data in URLs/logs, abuse of anonymous ingest (rate limits).

### Observability

- BusinessRules.API via `AddServiceDefaults()`; health check for `rulesdb` and the event bus.
- Metrics (meters registered in ServiceDefaults): `rules.snapshot.version`, `rules.snapshot.age_seconds`
  (per consumer), `rules.publish.count`, `experiments.running`, `experiments.exposures`,
  `pricing.quote.duration`, `pricing.total_mismatch.count` (409s), `tracking.events.accepted` /
  `.rejected`, `results.aggregation.duration`.
- Alerts: consumer snapshot age > 5 min; 409 rate > 2% of create-order calls over 15 min; tracking
  rejection rate > 5%.
- Traces/log scopes carry `order.id`, `experiment.key`, `variant.key`, `rules.snapshot.version` —
  never user ids or emails.

### Rollout

Phased; each phase deployable on its own and backward compatible. Defaults equal today's behaviour,
so phases 1–2 change nothing visible until a business admin publishes a rule.

1. **Foundation:** BusinessRules.API (rule types, versions, publish, audit, snapshot), shared client
   lib, `rulesdb`, AppHost wiring, Identity scopes/clients. Ordering server-side pricing (new Catalog
   client) + quote endpoint + create order v2 + `GetTotal` fix; **WebApp and ClientApp** move to quote
   + v2; v1 usage metric and publish guard. ADR-0008 written and accepted.
2. **Rule types:** PRC-001, PRM-001, SHP-001, ORD-001 per-order, ORD-013 incl. partial fulfilment
   (Ordering, OrderProcessor, WebApp/ClientApp display). Rules editable via API (Scalar) until phase 3.
3. **eShopAdmin:** onboard repo; rules UI (J8) with schedule, publish, history.
4. **Experiments:** model, validation, assignment, overrides in snapshot, exposure tracking,
   `OrderExperimentAssignment`, `OrderPricedIntegrationEvent`; eShopAdmin experiment UI.
5. **Tracking and results:** page-view/click tracking (web + mobile), ingest, aggregation, dashboards.

- Migration order: Ordering schema (additive + backfill) before code that reads it; OrderProcessor
  deploys after the `GracePeriodEndsAt` backfill.
- Rollback: disable a rule by publishing its default; stop experiments (everyone → control);
  WebApp can fall back to v1 create order (server-priced) if v2 misbehaves.
- ClientApp release cadence lags web: installed old app versions keep calling v1; the publish guard
  (see *Price consistency at checkout*) prevents price rules from silently diverging from what they show.
- Docs updated in the implementing PRs (Definition of Done): business-rules catalog (status of new
  rules → Adopted, mechanisms of ORD-001/ORD-013), service catalog, system context diagram,
  integration events (Proposed → Adopted), personas/journeys (J8 real; new **J10 Run an A/B test**),
  environments (new `rulesdb`, Key Vault HMAC secret).

## Acceptance criteria

- [ ] Given a published PRC-001 of 10% off catalog type "T-shirt", when a signed-in shopper views a
      T-shirt item, cart, and checkout, then each shows the reduced price and the created order charges
      exactly that price — without any deployment.
- [ ] Given a rule version with `EffectiveFrom` in the future, when that time passes, then quotes and new
      orders use it with no further action, and the audit log shows who published it.
- [ ] Given a WebApp that sends a tampered price or `expectedTotal`, when it creates an order, then the
      server charges its own computed price (v1) or returns 409 with a fresh quote (v2).
- [ ] Given a published change during checkout, when the shopper places the order, then they get a
      "prices changed" prompt and are never charged a total they weren't shown.
- [ ] Given experiment E1 (shipping threshold) and E2 (item-page promotion) both running on different
      rules, when a shopper visits, then they are assigned independently, consistently across pages,
      sessions and devices, and the order records both assignments.
- [ ] Given an anonymous shopper enrolled in a price experiment, when they browse catalog and item
      pages, then they see their variant's prices. When they sign in for the first time, then their
      variant is unchanged on that device and is the same on any other device after sign-in.
- [ ] Given a shopper who has not given tracking consent, when they browse and buy, then they still
      see their variant's prices, no tracking events are sent, and their order is excluded from
      results.
- [ ] Given an experiment whose overrides conflict with a running experiment under the override lock
      (same global rule, a different target kind on the same targeted rule, or overlapping target
      ids), when an
      admin schedules it, then it is rejected with 409.
- [ ] Given a running experiment, when an admin tries to edit its variants, then it is rejected; when
      they stop it, then all shoppers get published rules within 60 s.
- [ ] Given BusinessRules.API is down, when shoppers browse and check out, then T1 latency SLOs hold
      and the last snapshot (or defaults) is used.
- [ ] Given ORD-013 = `PartialFulfilment` resolved for an order with one out-of-stock line of three,
      when stock is rejected, then that line is marked unfulfillable, the order is repriced downward,
      moves to `StockConfirmed`, is paid for the reduced total, Catalog decrements only the two shipped
      items, and the shopper sees the dropped item on `/user/orders`.
- [ ] Given ORD-001 = 10 minutes for an order, when 5 minutes pass, then the order is still
      cancellable; at 10 minutes (+ polling interval) it moves to `AwaitingValidation`.
- [ ] Given tracked page views, clicks, and orders for a running experiment, when the admin opens
      results, then conversion, revenue, drop-off per funnel step, clicks per element, order-to-ship
      time, cancellation and partial-fulfilment rates are shown per variant with 95% CIs.
- [ ] Given a user erasure request, when it is processed, then their tracking events are deleted and
      their order outcomes remain pseudonymous.
- [ ] Given a `RulesEditor` (not publisher), when they try to publish or start an experiment, then
      the API returns 403.
- [ ] Existing E2E journeys J1, J2, J3, J5 stay green.

## Test strategy

Per [testing.md](../../standards/testing.md), TDD throughout.

- **Unit (MSTest + NSubstitute):** pricing calculator (evaluation order, best-promotion, allocation,
  rounding, invariants) — table-driven; assignment hashing (stability fixtures pinned to known
  outputs, distribution within ±1% over 100k keys); rule schema validation; experiment validation
  (overlap, immutability, weights); Order aggregate partial-fulfilment and repricing; `GetTotal`;
  results statistics against known fixtures.
- **Functional (WebApplicationFactory + Aspire Postgres/RabbitMQ):** BusinessRules.API admin/runtime/
  tracking endpoints incl. auth roles, 409s, rate limiting, snapshot 304; Ordering quote and create
  order v1/v2; event flows (snapshot published → consumer refresh; stock rejected → partial
  fulfilment → paid event content; order events → outcomes); OrderProcessor per-order grace period;
  migrations/backfill.
- **AppHost:** `rulesdb` and BusinessRules.API wiring.
- **E2E (Playwright):** existing J1–J3, J5 green; new: published promotion visible on item/cart and
  charged; price-changed prompt on checkout; partial-fulfilment display on `/user/orders`; tracking
  calls fire with `data-track` (network assertion). eShopAdmin: publish a rule; create/start/stop an
  experiment.
- **Resilience test:** stop BusinessRules.API; checkout still succeeds with last snapshot.

## Left to the builder

- Exact hash algorithm for bucketing (must be stable and identical across consumers).
- Snapshot serialization format and how consumers persist the last snapshot across restarts (if at all).
- Service-to-service auth mechanism for the snapshot endpoint (within security.md).
- Table partitioning and aggregation implementation (SQL vs. in-process) for tracking and results.
- eShopAdmin page layout and navigation (follow WebAppComponents and WCAG 2.2 AA).
- Whether the pricing calculator lives only in the shared lib or also behind a domain service in
  Ordering.Domain (keep it pure and shared).
- Names of endpoints/DTOs beyond those fixed above, following api-design.md.

## Open questions

- **Legal sign-off on tracking consent** (owner Q Brown). Settled: consent limits tracking only, not
  enrolment. Still open: whether the default is opt-in (likely for Quebec Law 25) or opt-out (CCPA),
  and the consent wording. Until sign-off, the builder implements **opt-in** (no tracking until
  consent) **(call)**, since that's the stricter option.

Decided during review (2026-11-03): anonymous shoppers are included in all experiments; consent
limits tracking only; no second-person approval for publishing.
