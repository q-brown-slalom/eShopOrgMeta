# 0009. One AppHost per repo, linked with external services

- Status: Proposed
- Date: 2026-11-04
- Deciders: Q Brown
- Scope: org
- Related: [spec](../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md),
  [plan (phase 3)](../plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments-phase-3.md),
  [ADR-0005](0005-deploy-to-azure-container-apps.md),
  [ADR-0008](0008-runtime-business-rules-service-and-in-house-experimentation.md)

## Context

[environments-and-deployment.md](../architecture/environments-and-deployment.md) left one question open:
will each repo have its own AppHost, or will a shared AppHost deploy all repos?

Phase 3 of the runtime-business-rules work adds **eShopAdmin**, the first repo beyond eShop. It is a
Blazor Web App for editing business rules (T3). At runtime it needs three things that live in eShop:

- Identity.API, to sign admins in (client `eshopadmin`).
- BusinessRules.API, for the admin API.
- Catalog.API, for type and brand names.

Today eShop's AppHost (`eShop/src/eShop.AppHost/AppHost.cs`) is the only AppHost. It already declares
`AddAzureContainerAppEnvironment("aca")` for [ADR-0005](0005-deploy-to-azure-container-apps.md).
Identity.API already registers the `eshopadmin` client when its `EShopAdminClient` setting is present
(`eShop/src/Identity.API/Configuration/Config.cs`), but nothing sets that setting yet.

Whatever we choose also applies to the other planned repo, eShopOffering.

## Options considered

1. **(A) One AppHost per repo.** eShopAdmin's AppHost declares eShop's services as `AddExternalService`
   resources whose URLs are parameters. eShop's AppHost gets a parameter for eShopAdmin's URL.
   Pros:    no cross-repo project paths; each repo builds and tests alone. Cons: the satellite
   needs eShop running to be exercised end to end; default ports are duplicated in two repos.
2. **(B) eShop's AppHost references `../../eShopAdmin` project paths.** Pros: one `aspire run` starts
   everything, with service discovery and `WaitFor`. Cons: eShop no longer builds without a sibling
   checkout at a fixed path; every satellite repo couples to eShop's release cadence; a T3 app becomes
   part of eShop's graph, which sits close to the T1 independence rule in ADR-0008.
3. **(C) A separate "platform" AppHost repo** that references every repo. Pros: neither product repo knows
   about the other. Cons: a new repo to own for what is one extra app; the same checkout-layout coupling
   as option 2; delays phase 3 for a decision with no second consumer yet.

## Decision

Option 1 (A). Each repo owns its AppHost. A repo that depends on eShop services declares them with
`AddExternalService` and takes their URLs from parameters.

Parameters and defaults (plan decision P3-D1). Defaults are eShop's launch-profile ports
(`eShop/src/Identity.API/Properties/launchSettings.json`,
`eShop/src/BusinessRules.API/Properties/launchSettings.json`,
`eShop/src/Catalog.API/Properties/launchSettings.json`):

| Parameter | Declared in | Default | Used for |
|-----------|-------------|---------|----------|
| `identity-url` | eShopAdmin AppHost | `https://localhost:5243` | OIDC authority (`identity-api`) |
| `businessrules-url` | eShopAdmin AppHost | `http://localhost:5230` | Admin API (`businessrules-api`) |
| `catalog-url` | eShopAdmin AppHost | `http://localhost:5222` | Type and brand lookups (`catalog-api`) |
| `eshopadmin-url` | eShop AppHost | `https://localhost:7250` | Identity.API's `EShopAdminClient` (redirect and client URIs) |

The `identity-url` default is eShop's `https` launch profile. When eShop runs with
`ESHOP_USE_HTTP_ENDPOINTS=1` (CI), it listens on `http://localhost:5223` instead, so the CI job
overrides `identity-url`.

## Consequences

- Positive: eShop and eShopAdmin build and test on their own; there are no cross-repo project paths;
  a new satellite repo follows the same recipe; eShopAdmin stays outside eShop's resource graph, and no T1
  service depends on eShopAdmin at runtime (Identity.API only registers its client URI through the
  `eshopadmin-url` parameter).
- Negative: ports are written twice and can drift, and the parameters are the only link between
  the repos; eShop's `eshopadmin-url` and eShopAdmin's own endpoint must be kept equal.
- **CI for a satellite repo** checks out eShop next to it, starts eShop's AppHost, then starts the
  satellite's AppHost and runs its tests against the external URLs. Both AppHosts run in the job.
- **Deployment (plan decision P3-D2).** Phase 3 does not deploy eShopAdmin: the Dev ACA pipeline does
  not exist yet for eShop either (status Proposed), so local and CI are the only environments in use.
  When it is deployed, eShopAdmin **must join eShop's Azure Container Apps environment**, because
  BusinessRules.API ingress is internal and is not reachable from outside it. The URL parameters then
  take the environment's internal FQDNs instead of the local defaults.
- Follow-ups (standards/docs to update):
  - Phase 3 (this plan): repo registry, service catalog, system context, NFR tiers, environments
    (parameters).
  - Answer the AppHost open question in
    [environments-and-deployment.md](../architecture/environments-and-deployment.md) and remove it.
  - When the Dev pipeline exists, design eShopAdmin's deployment (shared ACA environment, internal
    URLs) in its own plan.
  - Move to Accepted once the phase 3 AppHost wiring has merged in both repos.
