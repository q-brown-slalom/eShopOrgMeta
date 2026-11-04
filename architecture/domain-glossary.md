---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
source: eShop/src/Catalog.API/Model, eShop/src/Basket.API/Model, eShop/src/Ordering.Domain, eShop/src/BusinessRules.API/Model, eShop/src/eShop.BusinessRules.Client
---

# Domain glossary

Use these terms exactly in specs, code, and conversations. Add new terms when a spec introduces them.

| Term | Meaning | Owned by |
|------|---------|----------|
| Catalog item | A sellable product: name, description, price, picture, stock levels, AI embedding | Catalog.API |
| Catalog brand | Manufacturer/brand of a catalog item | Catalog.API |
| Catalog type | Product category of a catalog item | Catalog.API |
| Available stock | Units on hand; `RestockThreshold` / `MaxStockThreshold` bound reordering | Catalog.API |
| Basket (`CustomerBasket`) | A buyer's cart, keyed by buyer id, stored in Redis | Basket.API |
| Basket item | Line in a basket: product id/name, unit price, quantity | Basket.API |
| Order | Aggregate root for a purchase: items, address, buyer, status, payment | Ordering.Domain |
| Order item | Line in an order: product, unit price, discount, units | Ordering.Domain |
| Address | Value object: street, city, state, country, zip | Ordering.Domain |
| Order status | `Submitted`, `AwaitingValidation`, `StockConfirmed`, `Paid`, `Shipped`, `Cancelled` | Ordering.Domain |
| Grace period | Window after submission during which the order can be cancelled before validation | OrderProcessor |
| Buyer | Aggregate root for a customer in ordering, linked by identity GUID | Ordering.Domain |
| Payment method | Buyer's card details (card type, masked number, holder, expiration) | Ordering.Domain |
| Webhook subscription | A user's registration to receive callbacks for an event type (e.g. order paid/shipped, price changed) | Webhooks.API |
| Integration event | Async message between services via the event bus | All |
| Domain event | In-process event raised by an aggregate, handled via MediatR | Ordering |
| Business rule | A named, ID'd policy that governs behavior (e.g. ORD-001 grace period). Catalogued in [business-rules-catalog.md](business-rules-catalog.md); admin-editable ones will be managed by the planned admin app | Owning service / admin app (planned) |
| Offering | A brand storefront with its own web app (e.g. `eShopOffering`) that shares eShop platform services: catalog, basket, ordering, identity. Rules marked "per offering" (ORD-013, CAT-007) vary by offering | Offering web apps |
| Rule type | Code-defined contract for a business rule: ID, typed parameters, allowed ranges, built-in default (`IRuleType` in `eShop.BusinessRules.Client`; no production rule types until phase 2) | BusinessRules.API |
| Rule version | A published or draft value for a rule type, with effective-from/to dates and audit data (`RuleVersion`) | BusinessRules.API |
| Rules snapshot | Immutable, versioned set of published rule versions (and, later, running experiments) that services evaluate locally (`RulesSnapshot`; stored as `RulesSnapshotRecord`) | BusinessRules.API |
| Price quote | Server-computed prices for basket lines: base price, adjusted price, discounts, shipping fee, total (`PriceQuote`, `POST api/pricing/quote`) | Ordering.API |

## Proposed terms

Introduced by the Approved [runtime rules spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md). Status: **Proposed**; move rows into the table above when the work is implemented.

| Term | Meaning | Owned by |
|------|---------|----------|
| Promotion | A rule (PRM-001) that discounts a line or the order when its conditions are met | BusinessRules.API / Ordering.API (planned) |
| Shipping fee | Fee added to an order (SHP-001), optionally waived above a free-shipping threshold | Ordering.API (planned) |
| Stock shortfall policy | ORD-013 value fixed on each order: `AllOrNothing` or `PartialFulfilment` | Ordering.Domain (planned) |
| Partial fulfilment | Shipping the in-stock lines of an order and dropping the rest, repricing the order downward | Ordering.Domain (planned) |
| Unfulfillable order item | An order item dropped by partial fulfilment; kept for history, excluded from totals | Ordering.Domain (planned) |
| Experiment | An A/B test with 2-4 variants that override rule values for an audience over a time window | BusinessRules.API (planned) |
| Variant | One arm of an experiment; exactly one is the `control` (no overrides), others are treatments | BusinessRules.API (planned) |
| Experiment subject | The identity an assignment is keyed on (signed-in user GUID, or anonymous visitor id) | BusinessRules.API (planned) |
| Assignment | The variant a subject gets, computed deterministically from experiment key and subject key | eShop.BusinessRules.Client (planned) |
| Exposure | The first time a subject sees a page in an experiment's page list while enrolled | BusinessRules.API (planned) |
| Tracking event | A pseudonymous page-view, click, or exposure record used for experiment results | BusinessRules.API (planned) |
