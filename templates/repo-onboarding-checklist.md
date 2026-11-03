# New repo onboarding checklist

## Registration

- [ ] Row added to [repos/registry.md](../repos/registry.md) (name, purpose, owner).
- [ ] Services added to [architecture/service-catalog.md](../architecture/service-catalog.md) and
      [architecture/system-context.md](../architecture/system-context.md).
- [ ] Folder added to [eshop-org.code-workspace](../eshop-org.code-workspace).
- [ ] `specs/<repo>/` and `plans/<repo>/` created.
- [ ] CODEOWNERS entries added (here and in the new repo).

## Repo baseline (copy from eShop)

- [ ] `global.json`, `Directory.Build.props`, `Directory.Packages.props`, `.editorconfig`, `nuget.config`
- [ ] `.github/workflows/`: PR validation (build + test), markdownlint, Playwright (if UI)
- [ ] `.github/dependabot.yml`
- [ ] `.github/pull_request_template.md` from [templates/pull-request.md](pull-request.md)
- [ ] `.github/ISSUE_TEMPLATE/` from [templates/issue-feature.md](issue-feature.md) and [issue-bug.md](issue-bug.md)
- [ ] `AGENTS.md` stub pointing to eShopOrgMeta and listing repo-specific overrides only
- [ ] Branch protection on `main` (PR + required checks)

## Platform integration

- [ ] Uses `eShop.ServiceDefaults` (or equivalent) and an Aspire AppHost
- [ ] Identity.API client + scope registered
- [ ] Integration events documented in [architecture/integration-events.md](../architecture/integration-events.md)
