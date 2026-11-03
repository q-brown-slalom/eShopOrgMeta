---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/e2e/*.spec.ts, eShop/src/WebApp/Components/Pages, eShop/src/WebhookClient, eShop/src/ClientApp/Views
---

# Personas and user journeys

Specs name the persona(s) they serve and the journey(s) they change. Journeys marked **E2E**
are covered by Playwright and must stay green.

## Personas

| ID | Persona | Goals | Channel | Auth |
|----|---------|-------|---------|------|
| P1 | **Browsing shopper** | Discover and compare products | WebApp | Anonymous |
| P2 | **Signed-in shopper** | Buy and track orders (seed user: `alice`) | WebApp | Identity.API (`basket`, `orders` scopes) |
| P3 | **Mobile shopper** | Same as P2, on a phone | ClientApp (MAUI) via mobile-bff | Identity.API (PKCE) |
| P4 | **Integration partner** | Receive order and price events in their own systems | WebhookClient / own endpoint | Identity.API (`webhooks` scope) |
| P5 | **Business admin** (planned) | Change business rules safely, without a deployment | Admin web app | Admin scope + role (to be defined) |
| P6 | **Operator / SRE** | Keep services healthy and diagnose incidents | Aspire dashboard / Azure Monitor | Platform access |

## Journeys

| ID | Journey | Persona | Steps | Pages / views | Coverage |
|----|---------|---------|-------|---------------|----------|
| J1 | Browse catalog | P1, P2 | Home → filter by brand/type → paginate (9 per page) → item detail | `/`, `/item/{id}` | **E2E** `BrowseItemTest.spec.ts` |
| J2 | Manage cart | P2 | Add item → `/cart` → change quantity → remove | `/item/{id}`, `/cart` | **E2E** `CartTest.spec.ts` |
| J3 | Check out | P2 | Cart → Check out → address pre-filled → Place order → order list | `/cart`, `/checkout`, `/user/orders` | **E2E** `CheckoutTest.spec.ts` |
| J4 | Track order | P2 | `/user/orders` shows live status updates (Submitted → … → Shipped/Cancelled) | `/user/orders` | Manual |
| J5 | Sign in / out | P2 | `/user/login` → Identity.API → back; `/user/logout` | `/user/login`, `/user/logout` | **E2E** `login.setup.ts` |
| J6 | Mobile purchase | P3 | Login → Catalog → Item → Basket → Checkout → Order detail | `LoginView`, `CatalogView`, `CatalogItemView`, `BasketView`, `CheckoutView`, `OrderDetailView` | Unit (`ClientApp.UnitTests`) |
| J7 | Register a webhook | P4 | Add webhook (event type, URL, token) → grant check → list hooks → receive messages | `/add-webhook`, `/Home/RegisteredHooks`, `/Home/ReceivedMessages` | Functional (`Webhooks.FunctionalTests`) |
| J8 | Edit a business rule (planned) | P5 | Find rule → edit with validation → review/approve? → effective date → audit | eShopAdmin (pages defined in its first spec) | — |
| J9 | Diagnose a failed order | P6 | Find order → trace across services → inspect events/outbox | Aspire dashboard | — |

When a spec changes a journey, update this table and the matching e2e test in the same work.
