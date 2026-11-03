---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
---

# Repo registry

GitHub: eShop upstream is `dotnet/eShop`; org maintainer is `@q-brown-slalom`. Every repo in the org must be listed here; use the name in the `Repo`
column for `specs/<repo>/` and `plans/<repo>/`.

| Repo | URL | Status | Purpose | Tech | Owner | Consumes Meta via |
|------|-----|--------|---------|------|-------|-------------------|
| eShopOrgMeta | `{{TBD — remote not configured}}` | Active | Org memory, standards, specs, plans | Markdown, Copilot skills | Q Brown | — |
| eShop | <https://github.com/dotnet/eShop> | Active (forked/owned) | Core commerce platform: services, storefront, mobile BFF | .NET 10, Aspire, Postgres, Redis, RabbitMQ | Q Brown | Multi-root workspace |
| eShopOffering | — | Planned | Brand storefront for an additional offering, sharing eShop platform services | Expected: Blazor + WebAppComponents, Aspire | Q Brown | Multi-root workspace |
| eShopAdmin | — | Planned | Admin web app for editing business rules | Expected: Blazor, Identity.API admin scope | Q Brown | Multi-root workspace |

## Adding a repo

1. Follow [templates/repo-onboarding-checklist.md](../templates/repo-onboarding-checklist.md).
2. Add a row here and to [architecture/service-catalog.md](../architecture/service-catalog.md).
3. Add the folder to [eshop-org.code-workspace](../eshop-org.code-workspace).
4. Create `specs/<repo>/` and `plans/<repo>/`.
