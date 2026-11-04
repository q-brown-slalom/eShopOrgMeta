---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
source: eShop/tests/, eShop/e2e/, eShop/playwright.config.ts, eShop/global.json, eShopAdmin/e2e/*.spec.ts, eShopAdmin/playwright.config.ts, eShopAdmin/README.md
---

# Testing standard

## Approach

- **TDD is mandatory** for new behavior (Superpowers `test-driven-development`: red → green → refactor).
- Bug fixes start with a failing test that reproduces the bug.

## Layers

| Layer | Tooling | Location | Use for |
|-------|---------|----------|---------|
| Unit | MSTest (MSTest.Sdk, Microsoft.Testing.Platform), NSubstitute | `tests/<Service>.UnitTests` | Domain logic, handlers, validators |
| Functional | `WebApplicationFactory<Program>` + Aspire-hosted Postgres/RabbitMQ containers | `tests/<Service>.FunctionalTests` | Endpoint contracts, persistence, auth |
| AppHost | MSTest | `tests/eShop.AppHost.UnitTests` | Resource graph wiring |
| E2E | Playwright (TypeScript, Chromium) | `e2e/*.spec.ts` | User journeys in the web app |
| E2E (eShopAdmin) | Playwright (TypeScript, Chromium) | `eShopAdmin/e2e/*.spec.ts` | Journey J8 in the admin app. Needs eShop running (Identity.API and BusinessRules.API) and starts eShopAdmin's own AppHost |

## Conventions

- Test names describe behavior: `<Action><Scenario><Outcome>`, e.g. `CancelNonExistentOrderFails`.
- Mock only across boundaries (repositories, event bus, external HTTP); never mock the unit under test.
- Functional tests use a fixture class (`<Service>ApiFixture`) and fake auth middleware.
- E2E: `login.setup.ts` creates the authenticated state; tests run serially (shared basket state).
- Docker is required locally for functional and e2e tests.
- **No component-test library** (plan decision P3-D9). In eShopAdmin, logic lives in plain classes tested with
  MSTest, and pages are tested with Playwright through `getByRole`/`getByLabel` locators, which fail when a
  control has no accessible name. bUnit and axe-core are not on the [tech radar](tech-radar.md); adding them is a
  separate radar change. A manual WCAG 2.2 AA checklist runs before a UI phase is signed off.
- eShopAdmin E2E specs run serially (they publish shared rule versions), use the publisher and editor users from
  `PUBLISHER_USERNAME`, `EDITOR_USERNAME`, and `PASSWORD`, publish only the ORD-001 rule so the price guard never
  fires, and restore the built-in default afterwards (P3-D10).

## Commands

```powershell
dotnet test --solution eShop.Web.slnf
npm run test:e2e
```

eShopAdmin (run eShop first, see [onboarding](../ways-of-working/onboarding.md#5-run-eshopadmin-next-to-eshop-optional-for-rules-ui-work)):

```powershell
dotnet test --solution eShopAdmin.slnx
npm run test:e2e
```
