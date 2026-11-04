---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
source: eShop/src/eShop.AppHost/AppHost.cs, eShop/src/eShop.AppHost/Extensions.cs, eShop/src/Identity.API/Configuration/Config.cs
---

# System context

```mermaid
flowchart LR
  subgraph Clients
    Browser
    Mobile[ClientApp / HybridApp]
    WHC[WebhookClient]
    Admin[eShopAdmin]
  end

  Browser --> WebApp
  Browser --> Admin
  Mobile --> BFF[mobile-bff YARP]
  BFF --> Catalog & Ordering & Identity

  WebApp -->|HTTP| Catalog[Catalog.API]
  WebApp -->|gRPC| Basket[Basket.API]
  WebApp -->|HTTP| Ordering[Ordering.API]
  WebApp --> Identity[Identity.API]
  WHC --> Webhooks[Webhooks.API]

  Ordering -->|prices| Catalog
  Ordering -->|snapshot poll 60 s| Rules[BusinessRules.API]
  Rules -->|v1 usage, publish guard| Ordering
  Admin -->|admin v1| Rules
  Admin -->|type and brand names| Catalog
  Admin -->|sign-in| Identity

  Catalog --- CatalogDB[(catalogdb pgvector)]
  Identity --- IdentityDB[(identitydb)]
  Ordering --- OrderingDB[(orderingdb)]
  OrderProcessor --- OrderingDB
  Webhooks --- WebhooksDB[(webhooksdb)]
  Rules --- RulesDB[(rulesdb)]
  Basket --- Redis[(Redis)]

  Catalog & Basket & Ordering & OrderProcessor & PaymentProcessor & Webhooks & WebApp & Rules <-.-> Bus{{RabbitMQ eventbus}}
  Catalog -.optional.-> AI[OpenAI / Ollama]
```

- All resources are orchestrated by `eShop.AppHost` (Aspire). One Postgres server, one DB per service.
- Solid arrows = synchronous calls; dotted = integration events
  (see [integration-events.md](integration-events.md)).
- BusinessRules.API (T3) is wired in `AddBusinessRules` (`eShop/src/eShop.AppHost/Extensions.cs`).
  Ordering.API polls it in the background and never `WaitFor`s it, so checkout works while it is down
  ([ADR-0008](../decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md), Proposed).
  mobile-bff has no BusinessRules.API route.
- eShopAdmin (T3, its own repo and AppHost, [ADR-0009](../decisions/0009-per-repo-apphost-with-external-services.md),
  Proposed) is not part of eShop's AppHost. It reaches Identity.API, BusinessRules.API, and Catalog.API
  by URL parameters, and eShop's AppHost passes its URL to Identity.API (`eshopadmin-url`).
- Planned additions (offering web apps): see [repos/registry.md](../repos/registry.md). Update this
  diagram when they land.
