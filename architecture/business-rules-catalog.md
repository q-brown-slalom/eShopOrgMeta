---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
source: eShop appsettings.json files, Ordering.Domain, Ordering.API/Application/Validations, Catalog.API/Model, Webhooks.API
---

# Business rules catalog

The single list of business rules across the org. Specs that add or change a rule update this
file. The planned **business-rules admin app** will manage rules marked *Admin candidate*; the
first admin-app spec decides the final set.

**Columns:**

- **Mechanism:** `Config` means appsettings or an environment variable, so it can change without code. `Code` means it is hard-coded. `Seed` means it is seed data in the database.
- **Admin candidate:** `Yes` means it is a sensible runtime-editable rule. `No` means it is an invariant or technical setting. `TBD` means it needs a product decision.

## Ordering

| ID | Rule | Current value | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------------|----------------------|-------------|-----------------|
| ORD-001 | Grace period before a submitted order is validated (customer can cancel freely) | 1 minute | Config `BackgroundTaskOptions:GracePeriodTime` — `src/OrderProcessor/appsettings.json` | OrderProcessor | Yes |
| ORD-002 | Grace-period polling interval | 30 seconds | Config `BackgroundTaskOptions:CheckUpdateTime` — same file | OrderProcessor | No |
| ORD-003 | Order must contain at least one item | ≥ 1 | Code — `CreateOrderCommandValidator.cs` | Ordering.API | No |
| ORD-004 | Shipping address fields required (street, city, state, country, zip) | Required | Code — `CreateOrderCommandValidator.cs` | Ordering.API | Yes — allowed shipping countries only; required fields stay in code |
| ORD-005 | Card number length | 12–19 chars | Code — `CreateOrderCommandValidator.cs` | Ordering.API | No |
| ORD-006 | Card security number length | 3 chars | Code — `CreateOrderCommandValidator.cs` | Ordering.API | No |
| ORD-007 | Card must not be expired | expiration ≥ now (UTC) | Code — `CreateOrderCommandValidator.cs`, `PaymentMethod.cs` | Ordering.API, Ordering.Domain | No |
| ORD-008 | Accepted card types | Amex (1), Visa (2), MasterCard (3) | Seed — `src/Ordering.API/Infrastructure/OrderingContextSeed.cs` | Ordering.API | Yes |
| ORD-009 | Same product added twice merges into one line; units accumulate; the higher discount wins | — | Code — `Order.AddOrderItem` | Ordering.Domain | No |
| ORD-010 | Order item units must be > 0; discount cannot exceed line total | — | Code — `OrderItem.cs` | Ordering.Domain | No |
| ORD-011 | Cancellation not allowed once `Paid` or `Shipped` | — | Code — `Order.SetCancelledStatus` | Ordering.Domain | No |
| ORD-012 | Shipping only from `Paid` | — | Code — `Order.SetShippedStatus` | Ordering.Domain | No |
| ORD-013 | Stock shortfall policy (today: any rejected stock item cancels the whole order) | All-or-nothing | Code — `Order.SetCancelledStatusWhenStockIsRejected` | Ordering.Domain | Yes — per-offering policy: `AllOrNothing` \| `PartialFulfilment` (partial path needs a spec) |

## Catalog and inventory

| ID | Rule | Current value | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------------|----------------------|-------------|-----------------|
| CAT-001 | Restock threshold: stock level at which the item should be reordered | Per item (seed 10) | Data — `CatalogItem.RestockThreshold` | Not enforced — see G-05 | Yes |
| CAT-002 | Max stock threshold: stock added beyond it is capped | Per item (seed 200) | Data — `CatalogItem.MaxStockThreshold`, `AddStock` | Catalog.API | Yes |
| CAT-003 | Cannot remove stock when sold out or for ≤ 0 units; removal capped at available stock | — | Code — `CatalogItem.RemoveStock` | Catalog.API | No |
| CAT-004 | Stock is decremented when an order is paid | — | Code — `OrderStatusChangedToPaidIntegrationEventHandler` | Catalog.API | No |
| CAT-005 | A price change publishes `ProductPriceChangedIntegrationEvent` | — | Code — `CatalogApi.cs` (update item) | Catalog.API | No |
| CAT-006 | Default API page size | 10 (no server max) | Code — `PaginationRequest.cs` | Catalog.API | No — add a max page size (G-02) |
| CAT-007 | Storefront page size | 9 per page | Code — `WebApp` catalog page | WebApp | No — per-offering config set at deploy time |

## Payments

| ID | Rule | Current value | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------------|----------------------|-------------|-----------------|
| PAY-001 | Simulated payment outcome | Always succeeds (`true`) | Config `PaymentOptions:PaymentSucceeded` — `src/PaymentProcessor/appsettings.json` | PaymentProcessor | No (test-only) |

## Identity and sessions

| ID | Rule | Current value | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------------|----------------------|-------------|-----------------|
| IDN-001 | Access/identity token lifetime | 120 minutes | Config `TokenLifetimeMinutes` — `src/Identity.API/appsettings.json` | Identity.API | No (security) |
| IDN-002 | Long-lived token lifetime | 365 days | Config `PermanentTokenLifetimeDays` — same file | Identity.API | No (security) |
| IDN-003 | Web session cookie lifetime | 60 minutes | Config `SessionCookieLifetimeMinutes` — `src/WebApp/appsettings.json` | WebApp | No (security) |

## Webhooks

| ID | Rule | Current value | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------------|----------------------|-------------|-----------------|
| WHK-001 | Subscribable event types | `CatalogItemPriceChange`, `OrderShipped`, `OrderPaid` | Code — `Webhooks.API/Model/WebhookType.cs` | Webhooks.API | Yes — enable/disable existing types; new types need a spec |
| WHK-002 | Grant URL must be absolute, same origin as hook URL, and echo the `X-eshop-whtoken` token on OPTIONS | — | Code — `WebhookSubscriptionRequest.cs`, `GrantUrlTesterService.cs` | Webhooks.API | No (security) |

## Pricing and fulfilment (Proposed)

Status: **Proposed** by the Approved [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md).
That spec also moves ORD-001 (per order, 1–60 min) and ORD-013 (adds `PartialFulfilment`) to the runtime rule store. Update the rows above when it is implemented.

| ID | Rule | Default | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------|----------------------|-------------|-----------------|
| PRC-001 | Price adjustment: percent off or fixed unit price, by catalog item, type, brand, or all | None (catalog price) | Runtime rule (planned BusinessRules.API) | Ordering.API pricing | Yes |
| PRM-001 | Promotion: line or order discount when its conditions are met; the best single promotion per level wins | None | Runtime rule (planned) | Ordering.API pricing | Yes |
| SHP-001 | Shipping fee with optional free-shipping threshold | 0 | Runtime rule (planned) | Ordering.API pricing | Yes |

## Known gaps (input for future specs)

| ID | Gap |
|----|-----|
| G-01 | No pricing rules exist (promotions, tax, shipping fees). Discounts exist on order items but nothing sets them from business rules. Promotions and shipping fees are addressed by the [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md) (Approved); tax is not. |
| G-02 | Catalog API has no maximum page size, so very large pages can be requested. |
| G-03 | All rules are deploy-time config or code. There is no runtime rule store, audit trail, or effective-dating. The admin app needs all three. Addressed by the [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md) (Approved). **Phase 1 done:** BusinessRules.API now has the rule store (`RuleVersion`), audit trail (`AuditEntry`), effective-dating, and versioned snapshots (`eShop/src/BusinessRules.API/Model`; [ADR-0008](../decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md), Proposed). No production rule types are registered yet; PRC-001, PRM-001, SHP-001, ORD-001, and ORD-013 arrive in phase 2. Rules are editable through the API (Scalar) until eShopAdmin ships in phase 3. |
| G-04 | `GrantUrlTesterService` logs the webhook token in plain text, which violates [standards/security.md](../standards/security.md). |
| G-05 | `RestockThreshold` is stored, but no code compares stock to it. Nothing ever sets `OnReorder` to `true`; `AddStock` only resets it to `false`. The `CheckThreshold` mentioned in the `RemoveStock` docs doesn't exist. |
| G-06 | Card data is stored in violation of PCI DSS: plain-text card number in `PaymentMethod` (`orderingdb`), and card number plus security code in `ApplicationUser` (`identitydb`). Fix: [ADR-0007](../decisions/0007-tokenize-card-data.md). |
| G-07 | No retention, archival, or erasure mechanism exists. Fix: [ADR-0006](../decisions/0006-tiered-order-retention.md), [standards/data-retention.md](../standards/data-retention.md). |

## Retention rules

| ID | Rule | Value | Source |
|----|------|-------|--------|
| RET-001 | Active order window (full personal details) | 12 months | [ADR-0006](../decisions/0006-tiered-order-retention.md) |
| RET-002 | Archived order retention (pseudonymized financial fields) | 7 years from order date | ADR-0006 |
| RET-003 | Erasure request: pseudonymize orders immediately | — | ADR-0006 |
| RET-004 | Non-order data retention | See [data-retention.md](../standards/data-retention.md) | — |

All retention rules are **not admin-editable**; changes require an ADR and legal sign-off.
