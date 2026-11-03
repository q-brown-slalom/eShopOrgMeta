---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/*/Extensions/Extensions.cs (AddSubscription), eShop/src/PaymentProcessor/Program.cs
---

# Integration events

Transport: RabbitMQ (`eventbus`). Publishing uses the outbox (`IntegrationEventLogEF`).
Consumers are verified from `AddSubscription<...>()` registrations.

| Event | Publisher | Consumers |
|-------|-----------|-----------|
| `OrderStartedIntegrationEvent` | Ordering.API | Basket.API (clears basket) |
| `GracePeriodConfirmedIntegrationEvent` | OrderProcessor | Ordering.API |
| `OrderStatusChangedToSubmittedIntegrationEvent` | Ordering.API | WebApp |
| `OrderStatusChangedToAwaitingValidationIntegrationEvent` | Ordering.API | Catalog.API (stock check), WebApp |
| `OrderStockConfirmedIntegrationEvent` | Catalog.API | Ordering.API |
| `OrderStockRejectedIntegrationEvent` | Catalog.API | Ordering.API |
| `OrderStatusChangedToStockConfirmedIntegrationEvent` | Ordering.API | PaymentProcessor, WebApp |
| `OrderPaymentSucceededIntegrationEvent` | PaymentProcessor | Ordering.API |
| `OrderPaymentFailedIntegrationEvent` | PaymentProcessor | Ordering.API |
| `OrderStatusChangedToPaidIntegrationEvent` | Ordering.API | Catalog.API (decrement stock), Webhooks.API, WebApp |
| `OrderStatusChangedToShippedIntegrationEvent` | Ordering.API | Webhooks.API, WebApp |
| `OrderStatusChangedToCancelledIntegrationEvent` | Ordering.API | WebApp |
| `ProductPriceChangedIntegrationEvent` | Catalog.API | Webhooks.API |

## Order lifecycle (Ordering.Domain `Order`)

```mermaid
stateDiagram-v2
  [*] --> Submitted: CreateOrder
  Submitted --> AwaitingValidation: GracePeriodConfirmed
  AwaitingValidation --> StockConfirmed: OrderStockConfirmed
  AwaitingValidation --> Cancelled: OrderStockRejected
  StockConfirmed --> Paid: OrderPaymentSucceeded
  StockConfirmed --> Cancelled: OrderPaymentFailed
  Paid --> Shipped: ShipOrder
  Submitted --> Cancelled: CancelOrder
  AwaitingValidation --> Cancelled: CancelOrder
  StockConfirmed --> Cancelled: CancelOrder
  Shipped --> [*]
  Cancelled --> [*]
```

Rules enforced by the aggregate: cannot cancel once `Paid` or `Shipped`; can only ship from `Paid`.

When adding or changing an event, update this file in the same spec/PR.
