---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-04
source: eShop/.github/workflows/, eShop/.github/dependabot.yml, eShopAdmin/.github/workflows/pr-validation.yml, eShopAdmin/.github/workflows/playwright.yml, eShopAdmin/.github/workflows/markdownlint.yml
---

# CI/CD standard

## Required PR checks (GitHub Actions)

| Workflow | What it does | Reference |
|----------|--------------|-----------|
| `pr-validation.yml` | `dotnet build` + `dotnet test` on the solution filter (ubuntu-latest, SDK from `global.json`) | `eShop/.github/workflows/pr-validation.yml` |
| `playwright.yml` | Starts the app, runs `npm run test:e2e`, uploads report (30 days) | `eShop/.github/workflows/playwright.yml` |
| `markdownlint.yml` | Lints all Markdown | `eShop/.github/workflows/markdownlint.yml` |

- New repos copy these workflows as a baseline (see [templates/repo-onboarding-checklist.md](../templates/repo-onboarding-checklist.md)).
- Docs-only changes (`**.md`) skip build/test via `paths-ignore`.
- Dependabot enabled for NuGet and npm (recommended: add `github-actions`).
- Recommended addition: Spectral OpenAPI lint as a PR check for API repos.

### eShopAdmin workflows

eShopAdmin copies the baseline with these differences (`eShopAdmin/.github/workflows/`):

| Workflow | What it does | Reference |
|----------|--------------|-----------|
| `pr-validation.yml` | `dotnet build` + `dotnet test --solution eShopAdmin.slnx` (ubuntu-latest, SDK from `global.json`); skipped for Markdown-only changes | `eShopAdmin/.github/workflows/pr-validation.yml` |
| `playwright.yml` | Checks out eShop (repository variable `ESHOP_REPOSITORY`, branch `main`) into `eshop/`, starts eShop's AppHost in the background, waits for Identity.API and BusinessRules.API, then runs `npm run test:e2e` (Playwright starts eShopAdmin's AppHost) and uploads the report (30 days). See [ADR-0009](../decisions/0009-per-repo-apphost-with-external-services.md) | `eShopAdmin/.github/workflows/playwright.yml` |
| `markdownlint.yml` | Lints all Markdown with the repo's `.markdownlint.json`; runs on every pull request | `eShopAdmin/.github/workflows/markdownlint.yml` |

These workflows have not run yet because eShopAdmin has no remote yet.

## Deployment

Azure Container Apps via `aspire deploy` (accepted in [ADR-0005](../decisions/0005-deploy-to-azure-container-apps.md)).
Environments and promotion flow: [architecture/environments-and-deployment.md](../architecture/environments-and-deployment.md).
