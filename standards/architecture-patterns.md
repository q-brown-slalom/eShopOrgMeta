---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/Ordering.*, eShop/src/EventBus*, eShop/src/IntegrationEventLogEF
---

# Architecture patterns

## Service boundaries

- One service owns one database (Postgres DB per service on a shared server; Redis for Basket).
  **No cross-service DB access** — communicate via HTTP/gRPC or integration events.
- Synchronous calls for queries the UI needs now; asynchronous events for state propagation.

## Rich domains (DDD) — use when business rules are non-trivial

Reference: `Ordering.Domain`.

- Base types from `SeedWork`: `Entity`, `ValueObject`, `IAggregateRoot`, `IRepository<T>`, `IUnitOfWork`.
- Aggregates expose behavior methods (e.g. `Order.SetPaidStatus()`), private setters, private
  collections. Invariants are enforced inside the aggregate; invalid transitions throw a domain
  exception.
- Domain events (`INotification`) are raised in the aggregate and dispatched by MediatR on save.
- Simple CRUD services (Catalog-style) may use EF entities directly — don't add DDD ceremony
  without business rules.

## CQRS with MediatR

- Commands (`<Verb><Noun>Command`) change state; queries use a separate read path (`I<Name>Queries`).
- Pipeline behaviors, in order: `LoggingBehavior` → `ValidatorBehavior` (FluentValidation) →
  `TransactionBehavior`.
- One validator per command: `<Command>Validator`.

## Integration events

- Naming: past tense, `<Subject><Change>IntegrationEvent`, e.g.
  `OrderStatusChangedToPaidIntegrationEvent`. Handler: `<Event>Handler`.
- Publish through the **outbox** (`IntegrationEventLogEF`) in the same transaction as the state change.
- Handlers must be **idempotent** (events can be redelivered).
- Subscriptions are registered in `Extensions.cs` via `AddSubscription<TEvent, THandler>()`.
- Contracts are additive; breaking changes require a new event type and an ADR.
- Every event is listed in [architecture/integration-events.md](../architecture/integration-events.md).

## Front ends

- Blazor Web App + shared `WebAppComponents` for web storefronts. New offering web apps should
  reuse `WebAppComponents` where possible (see [repos/registry.md](../repos/registry.md)).
- Mobile goes through the YARP `mobile-bff`.
