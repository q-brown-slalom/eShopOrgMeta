# 0006. Tiered order retention

- Status: Accepted (pending legal sign-off by Q Brown)
- Date: 2026-11-03
- Deciders: Q Brown
- Scope: org

## Context

Orders are financial records subject to US and Canadian tax retention, and they also contain personal data
subject to CCPA/CPRA, PIPEDA, and Quebec Law 25. eShop has no retention or erasure mechanism today.

## Options considered

1. **Single period, then delete.** Simple, but keeps personal data far longer than needed.
2. **Tiered model.** Orders are active for 12 months, then archived as pseudonymized financial records for up to 7 years, then deleted. Meets tax obligations while minimizing personal data.
3. **Keep indefinitely.** Conflicts with storage-limitation principles.

## Decision

Option 2: active 12 months, archived to 7 years, then hard delete. Erasure requests pseudonymize
orders immediately and keep financial fields until deletion. Details:
[standards/data-retention.md](../standards/data-retention.md).

## Consequences

- Positive: compliant, with minimal personal data; one model for all repos.
- Negative: needs archival and erasure jobs in Ordering, plus an erasure workflow across Identity, Basket, Ordering, and Webhooks. Customers can see only 12 months of order history.
- Follow-ups:
  - Spec for retention jobs and the erasure workflow.
  - Confirm with the payment provider that chargeback evidence is sufficient after 12 months.
  - Legal sign-off.
