---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/eShop.AppHost/AppHost.cs, eShop/src/*
---

# Service catalog

| Service | Repo | Kind | Interface | Data | Publishes / Consumes events | Owner |
|---------|------|------|-----------|------|-----------------------------|-------|
| Identity.API | eShop | OIDC provider (Duende) | HTTP | Postgres `identitydb` | — | Q Brown |
| Catalog.API | eShop | Minimal API v1/v2 | HTTP | Postgres `catalogdb` (+pgvector) | Yes | Q Brown |
| Basket.API | eShop | gRPC service | gRPC | Redis | Yes | Q Brown |
| Ordering.API | eShop | Minimal API v1, DDD + CQRS | HTTP | Postgres `orderingdb` | Yes | Q Brown |
| OrderProcessor | eShop | Worker (grace period) | — | Postgres `orderingdb` | Yes | Q Brown |
| PaymentProcessor | eShop | Worker (simulated payments; Stripe planned, ADR-0007) | — | — | Yes | Q Brown |
| Webhooks.API | eShop | Minimal API | HTTP | Postgres `webhooksdb` | Consumes | Q Brown |
| WebhookClient | eShop | Web app (test client) | HTTP | — | — | Q Brown |
| WebApp | eShop | Blazor storefront | HTTP | — | Consumes | Q Brown |
| mobile-bff | eShop | YARP reverse proxy | HTTP | — | — | Q Brown |
| ClientApp / HybridApp | eShop | .NET MAUI clients | — | — | — | Q Brown |

Shared libraries (eShop): `eShop.ServiceDefaults`, `EventBus`, `EventBusRabbitMQ`,
`IntegrationEventLogEF`, `WebAppComponents`, `Shared`.

Event details: [integration-events.md](integration-events.md).
