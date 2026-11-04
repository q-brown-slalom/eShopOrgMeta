---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
source: eShop/src/eShop.AppHost/AppHost.cs, eShop/src/eShop.AppHost/Extensions.cs, eShop/src/*
---

# Service catalog

| Service | Repo | Kind | Interface | Data | Publishes / Consumes events | Owner |
|---------|------|------|-----------|------|-----------------------------|-------|
| Identity.API | eShop | OIDC provider (Duende) | HTTP | Postgres `identitydb` | — | Q Brown |
| Catalog.API | eShop | Minimal API v1/v2 | HTTP | Postgres `catalogdb` (+pgvector) | Yes | Q Brown |
| Basket.API | eShop | gRPC service | gRPC | Redis | Yes | Q Brown |
| Ordering.API | eShop | Minimal API v1/v2, DDD + CQRS; server-side pricing | HTTP | Postgres `orderingdb` | Yes | Q Brown |
| BusinessRules.API | eShop | Minimal API v1 (rule versions, publish, audit, snapshot); T3 | HTTP | Postgres `rulesdb`, schema `rules` | Publishes `BusinessRulesSnapshotPublishedIntegrationEvent` | Q Brown |
| OrderProcessor | eShop | Worker (grace period) | — | Postgres `orderingdb` | Yes | Q Brown |
| PaymentProcessor | eShop | Worker (simulated payments; Stripe planned, ADR-0007) | — | — | Yes | Q Brown |
| Webhooks.API | eShop | Minimal API | HTTP | Postgres `webhooksdb` | Consumes | Q Brown |
| WebhookClient | eShop | Web app (test client) | HTTP | — | — | Q Brown |
| WebApp | eShop | Blazor storefront | HTTP | — | Consumes | Q Brown |
| mobile-bff | eShop | YARP reverse proxy | HTTP | — | — | Q Brown |
| ClientApp / HybridApp | eShop | .NET MAUI clients | — | — | — | Q Brown |

Shared libraries (eShop): `eShop.ServiceDefaults`, `EventBus`, `EventBusRabbitMQ`,
`IntegrationEventLogEF`, `WebAppComponents`, `Shared`, `eShop.BusinessRules.Client` (snapshot cache,
pricing calculator, quote DTOs, service-token handler).

## Runtime business rules ([ADR-0008](../decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md), Proposed)

- **Ordering.API → Catalog.API:** Ordering.API reads base prices from `GET api/catalog/items/by`
  (`eShop/src/Ordering.API/Application/Pricing/CatalogPriceClient.cs`) and prices quotes and orders
  itself (`api/pricing/quote`, create order v2).
- **Ordering.API → BusinessRules.API:** Ordering.API prices from a locally cached rules snapshot. It loads
  the snapshot at startup, polls every 60 s, and refreshes on the event
  (`eShop/src/eShop.BusinessRules.Client/Snapshot/RulesSnapshotRefreshService.cs`). It never
  `WaitFor`s BusinessRules.API and never calls it while pricing (`eShop/src/eShop.AppHost/Extensions.cs`).
  Without a snapshot it uses built-in defaults (catalog price, no discount, no shipping fee).
- **BusinessRules.API → Ordering.API:** BusinessRules.API reads `GET api/usage/create-order-v1` before it
  publishes a price-affecting rule. If the read fails, the publish is refused (fail closed)
  (`eShop/src/BusinessRules.API/Services/V1ClientsPublishGuard.cs`).
- `GET api/rules/snapshot` is internal-only. mobile-bff has no BusinessRules.API route.
- Meters: `eShop.BusinessRules`, `eShop.BusinessRules.Client` (`rules.snapshot.version`,
  `rules.snapshot.age_seconds`), and `eShop.Ordering` (`eShop/src/eShop.ServiceDefaults/Extensions.cs`).

Event details: [integration-events.md](integration-events.md).
