---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-03
decisions: [ADR-0006, ADR-0007]
---

# Data retention and privacy

Jurisdictions: **United States and Canada**. The periods below are working assumptions until
Q Brown completes legal sign-off; this file is not legal advice.

Relevant regimes:

- US: IRS record-keeping, state privacy laws (CCPA/CPRA and others).
- Canada: CRA's 6 years from the end of the tax year, PIPEDA, and Quebec Law 25.
- PCI DSS for card data.

## Orders: tiered model ([ADR-0006](../decisions/0006-tiered-order-retention.md))

| Tier | Age (from order date) | What is kept | Who can see it |
|------|-----------------------|--------------|----------------|
| Active | 0–12 months | Full order: items, prices, status, buyer, shipping address, payment token reference | Customer (`/user/orders`), support |
| Archived | 12 months – 7 years | Financial fields only: order id, dates, items, unit prices, discounts, totals, tax, currency, status, payment provider reference, country/region (for tax). Buyer replaced by a pseudonymous id; name and street address removed | Finance and audit only |
| Deleted | > 7 years | Nothing. The record is hard-deleted | — |

- 7 years covers CRA (6 years from tax year end) and US IRS norms.
- Chargebacks can arrive up to ~540 days after purchase. After 12 months, dispute evidence relies on
  the payment provider reference and the archived fields. Confirm with the payment provider that
  this is sufficient.

## Erasure requests (CCPA/CPRA, PIPEDA, Law 25)

1. Delete the identity account (`identitydb`) and the basket.
2. Immediately pseudonymize all of the user's orders and their `Buyer` record, at any tier: remove
   name and street address, and replace the identity GUID with a pseudonymous id.
3. Keep the archived financial fields until the 7-year deletion date (legal obligation).
4. Remove the user's webhook subscriptions.
5. Log the request and its completion for audit, without personal data.

## Other data

| Data | Store | Retention |
|------|-------|-----------|
| Idempotency records (`ClientRequest`) | `orderingdb` | 30 days |
| Outbox entries (published) | each service DB | 7 days |
| Outbox entries (failed/unpublished) | each service DB | Until resolved; alert after 1 min (see NFRs) |
| Basket | Redis | 30 days since last activity |
| Webhook received-message log | WebhookClient / partner side | 90 days |
| Webhook subscriptions | `webhooksdb` | Until removed or the account is erased |
| Inactive identity accounts | `identitydb` | Flag after 3 years inactive; notify the user; delete 30 days after notice |
| Application logs | Azure Monitor | 30 days |
| Traces | Azure Monitor | 7 days |
| Backups | Postgres PITR | 7 days. Erasures re-apply on restore (keep an erasure log) |

## Card data ([ADR-0007](../decisions/0007-tokenize-card-data.md))

- Store only: payment provider token, card brand, last 4 digits, expiry month/year.
- Never store the full card number or the security code (CVV) in any database, log, event, or trace.

## Implementation notes

- Archival and deletion run as scheduled jobs per service, and must be idempotent and observable.
- Every new entity in a spec must state its retention tier. Unclassified data must not ship.
