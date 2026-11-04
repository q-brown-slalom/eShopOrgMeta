---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
---

# Non-functional requirements and SLOs

> Adopted 2026-11-03 as starting targets. Re-baseline against real telemetry once services are
> deployed, and record any changes with an ADR.

Specs must state which NFR tier each changed service falls in. They must also justify any deviation.

## Service tiers

| Tier | Services | Availability (monthly) | Error budget |
|------|----------|------------------------|--------------|
| T1 — revenue path | WebApp, Catalog.API, Basket.API, Ordering.API, Identity.API, mobile-bff | 99.9% | ~43 min/month |
| T2 — async processing | OrderProcessor, PaymentProcessor, Webhooks.API | 99.5% | ~3.6 h/month |
| T3 — internal/admin | WebhookClient, BusinessRules.API, eShopAdmin | 99.0% | ~7.2 h/month |

BusinessRules.API can be T3 because no T1 service depends on it at request time. Ordering.API prices
from a cached rules snapshot and falls back to built-in defaults when the service is down
([ADR-0008](../decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md), Proposed;
verified by `eShop/tests/Ordering.FunctionalTests/ResilienceTests.cs`).

## Latency (server-side, measured at the service)

| Operation type | p95 | p99 |
|----------------|-----|-----|
| Read (catalog browse, item, basket get) | < 300 ms | < 800 ms |
| Write (basket update, create order) | < 800 ms | < 2 s |
| Semantic search (AI embeddings) | < 1.5 s | < 3 s |
| Storefront page render (WebApp) | < 1 s | < 2.5 s |

## Async processing

| Flow | Target |
|------|--------|
| Integration event end-to-end delivery (publish → handler complete) | p95 < 5 s |
| Order `Submitted` → `Paid` (excluding grace period ORD-001) | p95 < 30 s |
| Webhook delivery after triggering event | p95 < 30 s, retried on failure |
| Outbox: unpublished events older than 1 min | Alert |

## Data

| Concern | Target |
|---------|--------|
| Backups (Postgres) | Daily; point-in-time restore 7 days |
| RPO / RTO (T1 data: orders, identity) | RPO ≤ 15 min, RTO ≤ 4 h |
| Basket (Redis) | Best-effort; loss is acceptable but must be rare |
| Retention | See [data-retention.md](data-retention.md) (orders: 12 months active, 7 years archived; logs 30 days; traces 7 days) |

## Capacity and scalability

- T1 services are stateless and scale horizontally; their state lives in Postgres or Redis.
- Baseline load to validate: **1,000 concurrent users** and **25 orders/min at peak** (per environment
  sized for Prod).

## Accessibility and UX

- Web UIs meet WCAG 2.2 AA.
- Supported browsers: the latest two versions of Chromium-based browsers, Firefox, and Safari.

## Security NFRs

See [security.md](security.md). In addition, dependency vulnerabilities rated High or Critical are fixed within 14 days.
