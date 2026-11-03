---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/Catalog.API/Model, eShop/src/Basket.API/Model, eShop/src/Ordering.Domain
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
