---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/Directory.Packages.props, eShop/src/eShop.AppHost/AppHost.cs
---

# Tech radar and approved dependencies

| Ring | Meaning |
|------|---------|
| **Adopt** | Default choice; use without asking. |
| **Trial** | Allowed in a scoped spec, with a named owner and an exit plan. |
| **Assess** | Explore only in a spike; production use needs an ADR. |
| **Hold** | Don't start new usage; migrate existing usage when touched. |

Adding anything new, or moving an entry to a different ring, requires an ADR. Exact versions
live in each repo's `Directory.Packages.props`, not here.

## Platform and runtime

| Item | Ring | Notes |
|------|------|-------|
| .NET 10 / C# (latest) | Adopt | Pinned via `global.json` |
| .NET Aspire 13.x (AppHost, ServiceDefaults, integrations) | Adopt | Local orchestration and deployment model |
| Azure Container Apps | Adopt | Deployment target ([ADR-0005](../decisions/0005-deploy-to-azure-container-apps.md)) |
| Kubernetes / AKS | Hold | Only revisit if ACA limits are hit |

## Data and messaging

| Item | Ring | Notes |
|------|------|-------|
| PostgreSQL + EF Core + Npgsql | Adopt | One DB per service |
| pgvector | Adopt | Catalog semantic search |
| Redis (StackExchange.Redis) | Adopt | Basket and caches only, never a system of record |
| RabbitMQ via `EventBusRabbitMQ` | Adopt | Integration events |
| Outbox (`IntegrationEventLogEF`) | Adopt | Required for publishing |
| Dapper | Trial | Read-side queries where EF is a measured bottleneck |
| Azure Service Bus | Assess | Possible managed replacement for RabbitMQ in Azure |

## Application frameworks

| Item | Ring | Notes |
|------|------|-------|
| ASP.NET Core Minimal APIs | Adopt | No MVC controllers for APIs |
| Asp.Versioning | Adopt | |
| gRPC (Grpc.AspNetCore) | Adopt | Internal low-latency calls |
| YARP | Adopt | BFFs |
| Blazor Web App + `WebAppComponents` | Adopt | Web front ends, including the planned admin app |
| .NET MAUI / Blazor Hybrid | Trial | Mobile clients |
| MediatR 13.x | Adopt (pinned) | Stay below the commercial-license versions; upgrading needs an ADR |
| FluentValidation | Adopt | |
| Duende IdentityServer | Adopt | Check license terms for production scale |
| Scalar (OpenAPI UI) | Adopt | |
| Swashbuckle | Hold | Replaced by built-in OpenAPI + Scalar |

## Observability and resilience

| Item | Ring | Notes |
|------|------|-------|
| OpenTelemetry (OTLP) | Adopt | Through ServiceDefaults |
| Microsoft.Extensions.Http.Resilience | Adopt | Standard resilience handler |
| Prometheus scraping endpoint | Assess | Commented out in ServiceDefaults |

## Payments

| Item | Ring | Notes |
|------|------|-------|
| Stripe (Elements/Checkout + Stripe.net SDK) | Adopt | Tokenized card handling ([ADR-0007](../decisions/0007-tokenize-card-data.md)) |
| Storing card numbers or security codes | Hold | Prohibited by PCI DSS (gap G-06) |

## AI

| Item | Ring | Notes |
|------|------|-------|
| Microsoft.Extensions.AI abstractions | Adopt | Keep providers swappable |
| Azure OpenAI / Microsoft Foundry | Trial | Toggled by `UseFoundry` |
| Ollama | Assess | Local development only |

## Testing and tooling

| Item | Ring | Notes |
|------|------|-------|
| MSTest (MSTest.Sdk, Microsoft.Testing.Platform) | Adopt | Default unit test framework |
| xUnit v3 | Hold | Present in CPM; don't use in new test projects |
| NSubstitute | Adopt | Default mocking library |
| Moq | Hold | Use NSubstitute instead |
| Playwright (TypeScript) | Adopt | E2E tests |
| Spectral | Adopt | OpenAPI lint |
| Superpowers skills | Adopt | [ADR-0002](../decisions/0002-adopt-superpowers.md) |

## Practices on Hold

| Item | Why |
|------|-----|
| Unpinned container image tags (e.g. `ankane/pgvector:latest` in AppHost) | Builds aren't reproducible; pin a version tag |
| Package versions in `.csproj` files | Breaks Central Package Management |
| Cross-service database access | Breaks service ownership |
