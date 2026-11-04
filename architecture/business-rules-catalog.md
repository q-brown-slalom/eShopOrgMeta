---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
source: eShop appsettings.json files, Ordering.Domain, Ordering.API/Application/Validations, Catalog.API/Model, Webhooks.API, eShop.BusinessRules.Client/Rules/Types, eShop.BusinessRules.Client/Pricing
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
| ORD-001 | Grace period before a submitted order is validated (customer can cancel freely) | 1 minute (default); 1–60 minutes | Runtime rule `ORD-001` (BusinessRules.API), resolved per order into `Order.GracePeriodEndsAt`; default 1 min — `src/eShop.BusinessRules.Client/Rules/Types/GracePeriodRuleType.cs`, `src/Ordering.API/Application/Rules/OrderFulfilmentRulesResolver.cs` | OrderProcessor (queries `GracePeriodEndsAt`) | Yes |
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
| ORD-013 | Stock shortfall policy: `AllOrNothing` cancels the whole order on any rejected stock item; `PartialFulfilment` drops the out-of-stock lines and reprices downward | `AllOrNothing` (default) | Runtime rule `ORD-013`, stored per order; `AllOrNothing` \| `PartialFulfilment` — `src/eShop.BusinessRules.Client/Rules/Types/StockShortfallPolicyRuleType.cs`, `Order.StockShortfallPolicy` and `Order.ApplyStockRejection` in `src/Ordering.Domain/AggregatesModel/OrderAggregate/Order.cs` | Ordering.Domain | Yes — per-offering policy |

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

## Pricing and fulfilment

Status: **Adopted** from the [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md) (phase 2).
The rules are evaluated in this order: PRC-001, then the best single line promotion per line, then the best single order promotion, then SHP-001 on the discounted subtotal. Nothing stacks within a level. Money is rounded to 2 decimals (`MidpointRounding.AwayFromZero`) per line. Ordering.API never calls BusinessRules.API on the request path; it reads the cached rules snapshot. An order's prices, grace period (ORD-001), and shortfall policy (ORD-013) are fixed when it is created. With no published version every rule uses its built-in default, so nothing changes for shoppers.

| ID | Rule | Default | Mechanism / location | Enforced by | Admin candidate |
|----|------|---------|----------------------|-------------|-----------------|
| PRC-001 | Price adjustment: percent off (0–90 %) or fixed unit price (> 0), by catalog item, type, brand, or all. The most specific target wins (item > type > brand > all); two entries with the same target are rejected. The adjusted unit price is never below 0.01 | None (catalog price) | Runtime rule `PRC-001` (BusinessRules.API) — `src/eShop.BusinessRules.Client/Rules/Types/PriceAdjustmentRuleType.cs`, `src/eShop.BusinessRules.Client/Pricing/PricingCalculator.cs` | Ordering.API pricing (`src/Ordering.API/Application/Pricing/OrderPricingService.cs`) | Yes |
| PRM-001 | Promotion: line or order discount when its conditions are met; label ≤ 60 characters. **Line:** an entry applies when its target matches and `minUnits` (if set) ≤ the line's units. The most specific applicable target is kept, then the highest discount wins; a tie goes to the first listed entry. Amount off is per unit and capped at the line total. **Order:** applies when the subtotal after line promotions ≥ `minSubtotal`; the highest discount wins. The amount is capped at that subtotal and split across lines in whole cents (`DiscountAllocation.Allocate`) | None | Runtime rule `PRM-001` (BusinessRules.API) — `src/eShop.BusinessRules.Client/Rules/Types/PromotionRuleType.cs`, `src/eShop.BusinessRules.Client/Pricing/PricingCalculator.cs`, `src/eShop.BusinessRules.Client/Pricing/DiscountAllocation.cs` | Ordering.API pricing | Yes |
| SHP-001 | Shipping fee (0–100) with optional free-shipping threshold (≥ 0), compared with the subtotal after discounts | 0 | Runtime rule `SHP-001` (BusinessRules.API) — `src/eShop.BusinessRules.Client/Rules/Types/ShippingFeeRuleType.cs`, `src/eShop.BusinessRules.Client/Pricing/PricingCalculator.cs` | Ordering.API pricing | Yes |

## Known gaps (input for future specs)

| ID | Gap |
|----|-----|
| G-01 | Tax is not modelled. (Promotions and shipping fees had no rules; the [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md) (Approved) addressed them.) **Phase 2 done:** PRC-001, PRM-001, and SHP-001 now set order prices and discounts from the rules snapshot (`eShop/src/eShop.BusinessRules.Client/Pricing/PricingCalculator.cs`, `eShop/src/Ordering.API/Application/Pricing`). |
| G-02 | Catalog API has no maximum page size, so very large pages can be requested. |
| G-03 | All rules are deploy-time config or code. There is no runtime rule store, audit trail, or effective-dating. The admin app needs all three. Addressed by the [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md) (Approved). **Phase 1 done:** BusinessRules.API now has the rule store (`RuleVersion`), audit trail (`AuditEntry`), effective-dating, and versioned snapshots (`eShop/src/BusinessRules.API/Model`; [ADR-0008](../decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md), Proposed). **Phase 2 done:** the production rule types PRC-001, PRM-001, SHP-001, ORD-001, and ORD-013 are registered (`eShop/src/eShop.BusinessRules.Client/Rules/Types`), and Ordering.API and OrderProcessor consume them. Rules are editable through the API (Scalar) until eShopAdmin ships in phase 3. |
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
