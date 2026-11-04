---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
source: eShop/e2e/*.spec.ts, eShop/src/WebApp/Components/Pages, eShop/src/WebhookClient, eShop/src/ClientApp/Views, eShopAdmin/e2e/*.spec.ts, eShopAdmin/src/eShopAdmin/Components/Pages, eShopAdmin/src/eShopAdmin/Auth/AdminPolicies.cs, eShopAdmin/src/eShopAdmin/Auth/AuthenticationExtensions.cs
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
| P5 | **Business admin** | Change business rules safely, without a deployment | eShopAdmin (Blazor Web App) | Identity.API `eshopadmin` (PKCE), scope `businessrules.admin`, roles `RulesEditor`/`RulesPublisher` |
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
| J8 | Edit a business rule | P5 | Rules list → history → new draft (typed form, server validation) → schedule (UTC) → publish (guard override if prompted) → audit log | eShopAdmin: `/rules`, `/rules/{id}`, `/rules/{id}/drafts/…`, `/rules/{id}/versions/{versionId}/publish`, `/audit` | **E2E** (eShopAdmin `e2e/*.spec.ts`); the guard override is covered by unit tests only |
| J9 | Diagnose a failed order | P6 | Find order → trace across services → inspect events/outbox | Aspire dashboard | — |

When a spec changes a journey, update this table and the matching e2e test in the same work.
