---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/eShop.AppHost/AppHost.cs
---

# System context

```mermaid
flowchart LR
  subgraph Clients
    Browser
    Mobile[ClientApp / HybridApp]
    WHC[WebhookClient]
  end

  Browser --> WebApp
  Mobile --> BFF[mobile-bff YARP]
  BFF --> Catalog & Ordering & Identity

  WebApp -->|HTTP| Catalog[Catalog.API]
  WebApp -->|gRPC| Basket[Basket.API]
  WebApp -->|HTTP| Ordering[Ordering.API]
  WebApp --> Identity[Identity.API]
  WHC --> Webhooks[Webhooks.API]

  Catalog --- CatalogDB[(catalogdb pgvector)]
  Identity --- IdentityDB[(identitydb)]
  Ordering --- OrderingDB[(orderingdb)]
  OrderProcessor --- OrderingDB
  Webhooks --- WebhooksDB[(webhooksdb)]
  Basket --- Redis[(Redis)]

  Catalog & Basket & Ordering & OrderProcessor & PaymentProcessor & Webhooks & WebApp <-.-> Bus{{RabbitMQ eventbus}}
  Catalog -.optional.-> AI[OpenAI / Ollama]
```

- All resources are orchestrated by `eShop.AppHost` (Aspire). One Postgres server, one DB per service.
- Solid arrows = synchronous calls; dotted = integration events
  (see [integration-events.md](integration-events.md)).
- Planned additions (offering web apps, business-rules admin app): see
  [repos/registry.md](../repos/registry.md). Update this diagram when they land.
