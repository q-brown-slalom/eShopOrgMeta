---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/tests/, eShop/e2e/, eShop/playwright.config.ts, eShop/global.json
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

## Conventions

- Test names describe behavior: `<Action><Scenario><Outcome>`, e.g. `CancelNonExistentOrderFails`.
- Mock only across boundaries (repositories, event bus, external HTTP); never mock the unit under test.
- Functional tests use a fixture class (`<Service>ApiFixture`) and fake auth middleware.
- E2E: `login.setup.ts` creates the authenticated state; tests run serially (shared basket state).
- Docker is required locally for functional and e2e tests.

## Commands

```powershell
dotnet test --solution eShop.Web.slnf
npm run test:e2e
```
