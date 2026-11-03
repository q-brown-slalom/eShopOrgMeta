# 0007. Tokenize card data via a payment provider

- Status: Accepted
- Date: 2026-11-03
- Deciders: Q Brown
- Scope: org

## Context

eShop stores card data in two places:

- `PaymentMethod` (`orderingdb`): the full card number in plain text.
- `ApplicationUser` (`identitydb`): the card number and the security code.

PCI DSS forbids storing the security code after authorization, and requires the card number
to be unreadable wherever it is stored. Payments are currently simulated (`PaymentOptions:PaymentSucceeded`).

## Options considered

1. **Tokenize via a payment provider.** Card data is entered into provider-hosted fields and never reaches our services, which minimizes PCI scope.
2. **Encrypt the card number at rest and remove the security code.** Card data is still in scope, and we would have to manage keys.
3. **Defer.** Not acceptable for production.

## Decision

Option 1, using **Stripe**: Stripe Elements or Checkout for card entry, and the Stripe .NET SDK in
PaymentProcessor. Persist only the provider token (Stripe PaymentMethod id), brand, last 4 digits,
and expiry. Remove card number and security code fields from `ApplicationUser` and `PaymentMethod`.
Replace the simulated PaymentProcessor with the Stripe integration before Prod.

## Consequences

- Positive: PCI scope reduced to the lowest self-assessment questionnaire tier (SAQ A); no card data to retain or erase.
- Negative: checkout UI changes (web and MAUI), data migrations, and a dependency on Stripe.
- Follow-ups: write a spec for the checkout, Ordering, and Identity changes, including Stripe webhooks for asynchronous payment results and how Stripe API keys are stored.
