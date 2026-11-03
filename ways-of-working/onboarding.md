---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/README.md, eShop/playwright.config.ts, eShop/e2e/login.setup.ts
---

# Onboarding (people and agents)

## 1. Install prerequisites

- .NET 10 SDK (the version in `eShop/global.json`)
- Docker Desktop or Podman, running
- Aspire CLI (`aspire --version`)
- Node.js LTS + npm (for Playwright)
- VS Code with C# Dev Kit, the Aspire extension, and GitHub Copilot; or Visual Studio with the ASP.NET workload
- Optional: .NET MAUI workload (mobile), Azure CLI (deploy / Foundry)

## 2. Clone and open

```powershell
git clone https://github.com/dotnet/eShop.git
git clone <eShopOrgMeta remote>   # side by side with eShop
code eShopOrgMeta/eshop-org.code-workspace
```

Agent setup for Superpowers: see the [README](../README.md#getting-started).

## 3. Run eShop locally

```powershell
cd eShop
aspire run         # opens the dashboard link in the console; Ctrl+C to stop
# or: aspire start / aspire ps / aspire stop
```

- Seed shopper: `alice`. Get the password from a teammate or the user seed; never commit it.
- Optional AI: `aspire secret set "Azure:SubscriptionId" <id>`, `aspire secret set "Azure:Location" eastus`,
  then `$env:UseFoundry = "true"; aspire run`.

## 4. Run tests

```powershell
dotnet test --solution eShop.Web.slnf
npm ci; npx playwright install chromium
$env:USERNAME1 = "alice"; $env:PASSWORD = "<password>"; npm run test:e2e
```

## 5. Read, in this order

1. [AGENTS.md](../AGENTS.md): how the memory is organized
2. [architecture/system-context.md](../architecture/system-context.md), [service-catalog.md](../architecture/service-catalog.md), [domain-glossary.md](../architecture/domain-glossary.md)
3. [ways-of-working/spec-driven-development.md](spec-driven-development.md)
4. [standards/](../standards/): skim all, then read in depth the ones for your area
5. [lessons-learned/](../lessons-learned/)

## 6. First task

Pick a `good first issue`. Run the Superpowers flow end to end on it: brainstorm, write a small spec, plan, use TDD, and open a PR. Ask your buddy (Q Brown) to review both the spec PR and the code PR.

## Troubleshooting

- Ports in use or Aspire file locks: see the `aspire-orchestration` skill in `eShop/.github/skills/`.
- E2E fails on HTTPS certificates in CI-like runs: set `ESHOP_USE_HTTP_ENDPOINTS=1`.
- Containers slow on first run: Postgres and RabbitMQ use persistent lifetimes and get reused afterwards.
