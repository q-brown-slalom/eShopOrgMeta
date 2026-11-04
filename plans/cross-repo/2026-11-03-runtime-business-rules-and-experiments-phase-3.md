---
title: "Runtime business rules and A/B experiments — phase 3 (eShopAdmin rules UI) implementation plan"
repo: "cross-repo"
status: Draft
approved_by: []
owner: "Q Brown"
created: 2026-11-04
last_reviewed: 2026-11-04
spec: "specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md"
scope: "Phase 3 (eShopAdmin) from the spec's Rollout section: onboard the eShopAdmin repo and ship the rules UI (J8) with schedule, publish, and history. Builds on phases 1–2 as merged."
---

# Runtime Business Rules — Phase 3 (eShopAdmin Rules UI) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Business admins (P5) find, edit, schedule, publish, and audit the five production rules in a
new eShopAdmin web app instead of through Scalar. Journey J8 becomes real.

**Architecture:** eShopAdmin is a new repo with its own Aspire AppHost. It references eShop's
Identity.API, BusinessRules.API, and Catalog.API as **external services** whose URLs are AppHost
parameters, so neither repo builds the other. The app is a Blazor Web App rendered **statically on the
server** (no interactive render modes). It signs in with the existing `eshopadmin` PKCE client, keeps
the access token in the auth cookie, and calls the BusinessRules.API v1 admin endpoints server-side.
BusinessRules.API stays the only authority for validation, roles, and the publish guard. The UI only
mirrors them.

**Tech Stack:** .NET 10, Aspire 13.6 (`AddExternalService`), Blazor Web App (static SSR + enhanced
forms), OpenID Connect against Duende IdentityServer 7.3, `Asp.Versioning.Http.Client`, MSTest +
NSubstitute, Playwright (TypeScript), GitHub Actions.

**Spec:** [specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md](../../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md)
(sections *Core model*, *Rule types in scope*, *Security*, *Rollout* item 3, *Left to the builder*:
eShopAdmin layout). Earlier phases:
[phase 1](2026-11-03-runtime-business-rules-and-experiments.md),
[phase 2](2026-11-03-runtime-business-rules-and-experiments-phase-2.md).

## Prerequisites (Definition of Ready)

- [x] Spec `status` is `Approved` (Q Brown, 2026-11-03).
- [ ] Reviewers accept the **Plan decisions** below (P3-D1 … P3-D11), including ADR-0009 (Task 1).
- [ ] Phase 2 merged to eShop `main` (the five rule types are registered in BusinessRules.API).
- [ ] Q Brown creates the empty GitHub repo `eShopAdmin` and the repository variable
      `ESHOP_REPOSITORY` (`owner/name` of the eShop fork). These are shared-system actions, so the agent
      doesn't do them.
- [ ] Branches: `feature/runtime-business-rules-phase-3` in eShop, `docs/runtime-business-rules-phase-3`
      in eShopOrgMeta, and `main` in the new local repo `AE-Immersion/eShopAdmin` (first commit) then
      `feature/rules-ui`. Use `using-git-worktrees` for the two existing repos.

## Global Constraints

- **No new third-party package** (spec *Constraints*). Every package eShopAdmin uses already exists in
  `eShop/Directory.Packages.props`. Copy the same versions into `eShopAdmin/Directory.Packages.props`.
  bUnit and axe-core are not on the [tech radar](../../standards/tech-radar.md), so don't use them.
- Tech radar *Adopt* only: Blazor Web App, Aspire 13.x, MSTest.Sdk + NSubstitute, Playwright (TS).
- `net10.0`, `<Nullable>enable</Nullable>`, warnings as errors, Central Package Management. Copy
  `global.json` from eShop (SDK `10.0.302`, `MSTest.Sdk` `4.0.2`, Microsoft.Testing.Platform runner).
- eShopAdmin is **T3** ([nfrs-and-slos.md](../../standards/nfrs-and-slos.md)). No T1 service may
  reference it.
- Web UI meets **WCAG 2.2 AA** ([nfrs-and-slos.md](../../standards/nfrs-and-slos.md#accessibility-and-ux)).
- Auth: Authorization Code + PKCE client `eshopadmin`, scope `businessrules.admin`, roles `RulesEditor`
  and `RulesPublisher`. Customer scopes are never requested (spec *Security*).
- BusinessRules.API enforces every rule. The UI never decides validity, roles, or the guard on its own.
  It hides actions the user can't take and shows the API's answer.
- No second-person approval. A publisher may publish their own draft (spec decision).
- Rule IDs, names, ranges, and parameter JSON exactly as phase 2 (P2-D1: camelCase, string enums,
  unknown members rejected, money ≤ 2 decimals).
- No PII in logs or traces. Log rule type ids, version numbers, and snapshot versions only, never `sub`
  or names ([security.md](../../standards/security.md)).
- eShopAdmin stores **no data** of its own: no database, only the auth cookie. No retention entry is
  needed ([data-retention.md](../../standards/data-retention.md)).
- Commits: Conventional Commits. Scope `admin` in eShopAdmin, the service scope in eShop, `docs` in
  eShopOrgMeta.

## Plan decisions (review before execution)

Each is a **(call)** that resolves something the spec leaves to the builder.

| # | Decision | Why |
|---|----------|-----|
| P3-D1 | **One AppHost per repo.** eShopAdmin's AppHost declares `identity-api`, `businessrules-api`, and `catalog-api` with `AddExternalService`. Their URLs are parameters `identity-url` (default `https://localhost:5243`), `businessrules-url` (`http://localhost:5230`), and `catalog-url` (`http://localhost:5222`). eShop's AppHost gains a parameter `eshopadmin-url` (default `https://localhost:7250`) that becomes Identity.API's `EShopAdminClient`. Recorded as **ADR-0009** (Task 1). | Answers the open question in [environments-and-deployment.md](../../architecture/environments-and-deployment.md). No cross-repo project paths. Each repo builds and tests alone. |
| P3-D2 | **No deployment in phase 3.** The Dev ACA pipeline doesn't exist yet for eShop either (status Proposed). ADR-0009 records that a deployed eShopAdmin must join eShop's ACA environment, because BusinessRules.API ingress is internal. | Avoids designing deployment twice. Local and CI are the only environments in use. |
| P3-D3 | **Static server-side rendering only** (no `InteractiveServer`/`InteractiveWebAssembly`). Lists of entries grow and shrink through submit buttons with named form handlers. | `HttpContext` (and so the saved access token) is always available, which it isn't inside interactive circuits. Plain HTML forms are the simplest path to WCAG 2.2 AA. |
| P3-D4 | **Roles reach the UI through a new `roles` identity resource** (claim `role`) in Identity.API, allowed on `eshopadmin`. eShopAdmin requests scope `roles`, reads `role` from userinfo, and sets `RoleClaimType = "role"`. | Today `role` is only in the access token (scope `businessrules.admin`'s `UserClaims`). The UI needs it to hide *Publish* from editors. |
| P3-D5 | **The session lasts as long as the access token.** The `eshopadmin` client has no `offline_access`, so there are no refresh tokens. The cookie expires 60 s before the access token and isn't sliding. A 401 from the API triggers a new sign-in that returns to the same URL. Unsaved form input is lost in that case (forms are short). | Keeps the client public (PKCE, no secret) and needs no token store. |
| P3-D6 | **Typed forms for the five known rule types.** eShopAdmin keeps its own copies of the API DTOs and parameter shapes, as ClientApp does (phase 1 D9). A rule type the app doesn't know is listed **read-only**. Server validation errors carry a JSON path (phase 2 Task 2), and the app maps them to form fields. | Business users must not edit JSON (spec anti-goal: no free-form rules). A shared NuGet package would need a feed, which is out of scope. The server stays authoritative, so drift fails safely. |
| P3-D7 | **All times are UTC**, entered with `datetime-local` and labelled "(UTC)". | Static SSR doesn't know the browser's time zone, and admins work across US and Canada zones. One unambiguous zone matches the audit log. |
| P3-D8 | **Targets:** Type and Brand are dropdowns filled from Catalog.API `GET api/catalog/catalogtypes` and `catalogbrands` (`api-version=1.0`, anonymous). Item is a numeric id, and the version page shows its name from `GET api/catalog/items/by?ids=`. If Catalog is down, every target becomes a numeric input with a notice. | Admins pick names, not ids, for the common cases, without a new search API. Catalog is T1, but the admin app must still work without it. |
| P3-D9 | **No component-test library.** Logic lives in plain classes tested with MSTest. Pages are tested with Playwright through `getByRole`/`getByLabel` locators, which fail when a control has no accessible name. A manual WCAG checklist runs at exit. | bUnit and axe-core aren't on the radar. Proposing them is a separate radar change. |
| P3-D10 | **E2E users and data:** `alice` = publisher and `bob` = editor (phase 1 D8), from env `PUBLISHER_USERNAME`, `EDITOR_USERNAME`, and `PASSWORD`. Specs publish **ORD-001** only, so the price guard never fires nondeterministically, and they restore the built-in default in `afterAll`. Drafts left behind are acceptable because the API has no discard. | Keeps E2E deterministic against a shared local database. The guard UX is covered by unit tests (Task 4) instead. |
| P3-D11 | **Discarding drafts is out of scope.** BusinessRules.API has no endpoint for it. *New draft* is allowed while other drafts exist. | Avoids an API change the spec doesn't ask for. Add it later if drafts pile up. |

### UI copy (exact)

| Key | Text |
|-----|------|
| Access denied | "You don't have access to eShopAdmin. Ask an administrator for the RulesEditor or RulesPublisher role." |
| Default value | "Built-in default" |
| Unknown rule type | "This rule type isn't supported by this version of eShopAdmin. You can view it but not edit it." |
| Catalog unavailable | "Catalog names are unavailable right now. Enter ids instead." |
| Past start time | "A start time in the past takes effect as soon as the version is published." |
| Price warning (publish) | "This rule changes prices. Shoppers see the new prices within 60 seconds of it taking effect." |
| Published | "Published. Rules snapshot {SnapshotVersion}. Shoppers see the change within 60 seconds of it taking effect." |
| Guard (409 `v1-clients-active`) | "Older app versions placed orders in the last 7 days and may show out-of-date prices." + the API's `detail` |
| Guard reason label | "Reason for publishing anyway (required, up to 500 characters)" — button "Publish anyway" |
| Guard unavailable (503) | "Publishing is paused because app usage couldn't be checked. Try again in a few minutes." |
| Concurrent publish | "Another publish happened at the same time. Reload this page and try again." |
| Other 409 | "This version can't be changed any more: {detail}" |

## Review Focus

Spec-implied conditions no task's main tests exercise, most likely to bite first. Each has a test in the
task named.

1. **The access token expires or is revoked while an admin is mid-task.** Expect a fresh sign-in that
   returns to the same page, not a 500 or a silent failure. → Task 3, `SessionEndsBeforeAccessToken`;
   Task 4, `UnauthorizedThrowsReauthenticationRequired` and `MiddlewareChallengesWithReturnUrl`.
2. **Server validation errors on nested fields** (`entries[2].value`, `linePromotions[0].label`). Expect
   each message next to the matching input and anything unmappable in the summary, never dropped. →
   Task 5, `MapsJsonPathErrorsToFormFields`, `UnmappableErrorsGoToSummary`.
3. **Parameters the app can't map** (a new rule type in eShop, or a published value whose shape differs
   from the app's copy). Expect a read-only view with the raw JSON, no crash, and no way to save a lossy
   edit. → Task 5, `FromParametersReturnsNullForUnrecognisedShape`; Task 6,
   `UnknownRuleTypeIsListedReadOnly`.
4. **Effective window input:** a start in the past, an end not after the start, or an empty end. Expect
   the past-start notice, a field error before any API call, and "no end" for empty. → Task 7,
   `WindowRejectsEndNotAfterStart`, `WindowFlagsPastStart`, `EmptyEndMeansOpenEnded`.
5. **An editor reaching *Publish* by URL, or a double-submitted publish.** Expect the access-denied page
   for the editor, and "can't be changed any more" (not an error page) for the second submit. → Task 4,
   `PublishOfNonDraftMapsToConflict`; Task 8, E2E `editor cannot publish`.

---

## File map

| Repo | Path | Responsibility |
|------|------|----------------|
| eShopOrgMeta | `decisions/0009-…md`, `repos/registry.md`, `eshop-org.code-workspace`, `.github/CODEOWNERS`, `architecture/*`, `standards/*` | ADR, onboarding, docs |
| eShop | `src/Identity.API/Configuration/Config.cs`, `src/eShop.AppHost/{AppHost,Extensions}.cs` | `roles` resource; `eshopadmin-url` → `EShopAdminClient` |
| eShopAdmin | repo root (`global.json`, `Directory.*.props`, `.editorconfig`, `nuget.config`, `aspire.config.json`, `eShopAdmin.slnx`, `AGENTS.md`, `.github/`) | Baseline copied from eShop |
| eShopAdmin | `src/eShopAdmin.AppHost/` | External services, admin project |
| eShopAdmin | `src/eShopAdmin.ServiceDefaults/` | Aspire template defaults (OTel, health, resilience, service discovery) |
| eShopAdmin | `src/eShopAdmin/Auth/` | OIDC, policies, token forwarding, re-authentication |
| eShopAdmin | `src/eShopAdmin/BusinessRules/` | API client, DTO copies, result types |
| eShopAdmin | `src/eShopAdmin/Catalog/` | Type/brand/item name lookups |
| eShopAdmin | `src/eShopAdmin/Rules/` | Form models, JSON mapping, error mapping, summaries |
| eShopAdmin | `src/eShopAdmin/Components/` | Layout and pages |
| eShopAdmin | `tests/eShopAdmin.UnitTests/`, `tests/eShopAdmin.AppHost.Tests/` | MSTest |
| eShopAdmin | `e2e/`, `playwright.config.ts`, `package.json` | Playwright journeys |

---

### Task 1: ADR-0009 and onboarding registration (eShopOrgMeta)

**Files:**
- Create: `decisions/0009-per-repo-apphost-with-external-services.md` (from [templates/adr.md](../../templates/adr.md))
- Modify: `decisions/README.md` (index row), `repos/registry.md` (eShopAdmin → `Active`, tech "Blazor Web App (static SSR), Aspire", URL once created), `eshop-org.code-workspace` (folder `{ "name": "eShopAdmin", "path": "../eShopAdmin" }`), `.github/CODEOWNERS` (no change needed if `*` covers it; confirm), `architecture/service-catalog.md` (eShopAdmin row: T3, no data store, calls BusinessRules.API admin v1, Catalog.API, Identity.API), `architecture/system-context.md` (diagram node + edges), `standards/nfrs-and-slos.md` line 20 ("business-rules admin app (planned)" → "eShopAdmin"), `architecture/environments-and-deployment.md` (answer the AppHost open question; add the `eshopadmin-url`, `identity-url`, `businessrules-url`, `catalog-url` parameters to *Configuration differences*)

**Interfaces:**
- Produces: ADR-0009 number and the parameter names that Tasks 2, 3, and 10 use.

- [ ] **Step 1:** Write ADR-0009. Context = the open question in environments-and-deployment.md and the eShopAdmin rollout. Options: (A) one AppHost per repo with `AddExternalService` + URL parameters; (B) eShop's AppHost references `../../eShopAdmin` project paths; (C) a separate "platform" AppHost repo. Decision A, with P3-D1/P3-D2 as consequences: CI for a satellite repo checks out eShop and runs both AppHosts; a deployed satellite joins eShop's ACA environment; parameter defaults are eShop's launch-profile ports. Status `Proposed`, scope `org`, related: spec, this plan, ADR-0005.
- [ ] **Step 2:** Make the registration and doc edits listed above. Update `last_reviewed` on each touched file and cite eShop file paths as sources.
- [ ] **Step 3:** Check every relative link in the touched files resolves (open each target). Expected: no broken links.
- [ ] **Step 4: Commit** `docs(adr): propose ADR-0009 per-repo AppHost and register eShopAdmin`

### Task 2: eShop — `roles` identity resource and `eshopadmin-url`

**Files:**
- Modify: `src/Identity.API/Configuration/Config.cs` (`GetResources`; `eshopadmin` `AllowedScopes`, ≈ line 236)
- Modify: `src/eShop.AppHost/Extensions.cs` (new `AddEShopAdminClient`), `src/eShop.AppHost/AppHost.cs` (call it after `identityApi` is declared)
- Test: `tests/Application.UnitTests/IdentityConfigurationTests.cs`, `tests/eShop.AppHost.UnitTests/EShopAdminClientTests.cs` (new; same model-building pattern as `BusinessRulesResourceTests`)

**Interfaces:**
- Produces:
  - `new IdentityResource("roles", "Your roles", new[] { JwtClaimTypes.Role })` in `GetResources()`
  - `eshopadmin.AllowedScopes` = `openid`, `profile`, `roles`, `businessrules.admin` (still no customer scopes)
  - `public static IResourceBuilder<ParameterResource> AddEShopAdminClient(this IDistributedApplicationBuilder builder, IResourceBuilder<DotnetProjectResource> identityApi)`: `AddParameter("eshopadmin-url", "https://localhost:7250")` (not secret), then `identityApi.WithEnvironment("EShopAdminClient", parameter)`. CI overrides it with `Parameters__eshopadmin-url=http://localhost:5250`.

- [ ] **Step 1: Write failing tests**
  - `RolesIdentityResourceCarriesRoleClaim`: `GetResources()` has `roles` whose `UserClaims` is exactly `["role"]`
  - Update `EShopAdminClientRequiresPkceAndAdminScopeOnly`: allowed scopes are exactly `openid, profile, roles, businessrules.admin`
  - `IdentityApiReceivesEShopAdminClientUrl`: `identity-api` has an environment callback that sets `EShopAdminClient`, and parameter `eshopadmin-url` exists and isn't secret
- [ ] **Step 2: Run** `dotnet test --project tests/Application.UnitTests --filter "FullyQualifiedName~IdentityConfigurationTests"` and `dotnet test --project tests/eShop.AppHost.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both, plus `dotnet build eShop.Web.slnf`. Expected: PASS, 0 warnings.
- [ ] **Step 5: Commit** `feat(identity): add roles identity resource and wire the eShopAdmin client URL`

### Task 3: eShopAdmin — repo baseline, AppHost, sign-in, access policies

**Files (all in the new `eShopAdmin` repo):**
- Create from eShop: `global.json`, `Directory.Build.props`, `Directory.Packages.props` (only the packages below, same versions), `.editorconfig`, `nuget.config`, `.gitignore`, `aspire.config.json` (`appHost.path` = `src/eShopAdmin.AppHost/eShopAdmin.AppHost.csproj`, Aspire `13.6.0`)
- Create from eShopOrgMeta templates: `.github/pull_request_template.md`, `.github/ISSUE_TEMPLATE/{feature,bug}.md`, `.github/dependabot.yml` (nuget + npm + github-actions, weekly), `AGENTS.md` (points to eShopOrgMeta `AGENTS.md`; lists no overrides), `README.md` (run eShop first, then `aspire run`)
- Create: `eShopAdmin.slnx`
- Create: `src/eShopAdmin.ServiceDefaults/` (Aspire 13.6 service-defaults template; register no extra meters)
- Create: `src/eShopAdmin.AppHost/{eShopAdmin.AppHost.csproj,AppHost.cs,Properties/launchSettings.json}`. Dashboard `https://localhost:19890;http://localhost:18850`, OTLP `https://localhost:18078`, resource service `https://localhost:19078`, so it runs next to eShop's dashboard.
- Create: `src/eShopAdmin/{eShopAdmin.csproj,Program.cs,Properties/launchSettings.json}` (profiles `http` = `http://localhost:5250`, `https` = `https://localhost:7250;http://localhost:5250`)
- Create: `src/eShopAdmin/Auth/{AdminPolicies,SessionLifetime,AuthenticationExtensions}.cs`
- Create: `src/eShopAdmin/Components/{App.razor,Routes.razor,_Imports.razor,Layout/MainLayout.razor,Pages/Home.razor,Pages/AccessDenied.razor}`
- Create: `tests/eShopAdmin.UnitTests/` (MSTest.Sdk + NSubstitute), `tests/eShopAdmin.AppHost.Tests/` (MSTest.Sdk)
- Create: `package.json` (`@playwright/test` at eShop's version, script `test:e2e`), `playwright.config.ts`, `e2e/login.setup.ts`, `e2e/SignInTest.spec.ts`
- Test: `tests/eShopAdmin.UnitTests/Auth/{AdminPoliciesTests,SessionLifetimeTests,RouteAuthorizationTests}.cs`, `tests/eShopAdmin.AppHost.Tests/AppHostTests.cs`

Packages (all in eShop CPM): `Aspire.Hosting.AppHost` SDK, `Microsoft.AspNetCore.Authentication.OpenIdConnect`, `Asp.Versioning.Http.Client`, `Microsoft.Extensions.Http.Resilience`, `Microsoft.Extensions.ServiceDiscovery`, the `OpenTelemetry.*` set the template needs, `MSTest`, `NSubstitute`.

**Interfaces:**
- Produces:
  - AppHost (`ESHOP_USE_HTTP_ENDPOINTS=1` selects the `http` profile, as in eShop):

    ```csharp
    var identityUrl = builder.AddParameter("identity-url", "https://localhost:5243");
    var identity = builder.AddExternalService("identity-api", identityUrl);
    var businessRules = builder.AddExternalService("businessrules-api", builder.AddParameter("businessrules-url", "http://localhost:5230"));
    var catalog = builder.AddExternalService("catalog-api", builder.AddParameter("catalog-url", "http://localhost:5222"));
    var admin = builder.AddProject<Projects.eShopAdmin>("eshopadmin", launchProfileName)
        .WithExternalHttpEndpoints()
        .WithReference(identity).WithReference(businessRules).WithReference(catalog)
        .WithEnvironment("IdentityUrl", identityUrl);
    admin.WithEnvironment("CallBackUrl", admin.GetEndpoint(launchProfileName));
    ```

  - `public static class AdminPolicies { const string Access = "Admin.Access"; const string Publish = "Admin.Publish"; const string EditorRole = "RulesEditor"; const string PublisherRole = "RulesPublisher"; static void Configure(AuthorizationOptions options); }`. `Access` = authenticated + role Editor or Publisher. `Publish` = authenticated + role Publisher. `FallbackPolicy` = `Access`, so a new page is denied by default.
  - `public static class SessionLifetime { public static DateTimeOffset ExpiresAt(DateTimeOffset now, int accessTokenExpiresInSeconds); }` = `now + max(0, expiresIn − 60) s` (P3-D5).
  - `public static IHostApplicationBuilder AddAdminAuthentication(this IHostApplicationBuilder builder)`:
    - Cookie: `SlidingExpiration = false`, `HttpOnly`, `SecurePolicy = Always` (except the `http` profile), `SameSite = Lax`, `AccessDeniedPath = "/access-denied"`.
    - OIDC: `Authority = IdentityUrl`, `ClientId = "eshopadmin"`, no secret, `ResponseType = "code"`, `UsePkce = true`, `SaveTokens = true`, `GetClaimsFromUserInfoEndpoint = true`, `MapInboundClaims = false`. Scopes `openid profile roles businessrules.admin`. `ClaimActions.MapJsonKey("role", "role")`. `TokenValidationParameters.NameClaimType = "name"`, `RoleClaimType = "role"`. `RequireHttpsMetadata` only outside Development. `OnTokenValidated` sets `Properties.ExpiresUtc = SessionLifetime.ExpiresAt(now, TokenEndpointResponse.ExpiresIn)` and `IsPersistent = false`.
    - Endpoints `GET /authentication/login?returnUrl=` (challenge; `returnUrl` must be local, otherwise `/`) and `POST /authentication/logout` (antiforgery-protected; signs out of both schemes).
  - Layout: skip link, `<header>` with app name, nav (*Rules*, *Audit log*), the user's `name`, and a sign-out form. Every page sets a unique `<PageTitle>` "… · eShopAdmin".
  - `/access-denied` (`[AllowAnonymous]`) shows the access-denied copy.

- [ ] **Step 1: Write failing tests**

```csharp
[TestMethod] [DataRow("RulesEditor", true)] [DataRow("RulesPublisher", true)] [DataRow(null, false)]
public async Task AccessPolicyRequiresARulesRole(string? role, bool allowed)
    => Assert.AreEqual(allowed, (await Authorize(User(role), AdminPolicies.Access)).Succeeded);

[TestMethod] public async Task PublishPolicyRejectsEditor()
    => Assert.IsFalse((await Authorize(User("RulesEditor"), AdminPolicies.Publish)).Succeeded);

[TestMethod] public void SessionEndsBeforeAccessToken()                   // Review Focus 1
    => Assert.AreEqual(Now.AddSeconds(3540), SessionLifetime.ExpiresAt(Now, 3600));

[TestMethod] public void SessionLifetimeNeverNegative()
    => Assert.AreEqual(Now, SessionLifetime.ExpiresAt(Now, 30));
```

  `User(role)` builds a `ClaimsPrincipal` with authentication type `"Test"` and `role` claims, and `Authorize` uses a `DefaultAuthorizationService` from a `ServiceCollection` with `AddAuthorization(AdminPolicies.Configure)`.
  - `EveryRoutablePageRequiresAuthorizationExceptAccessDenied`: reflect over the app assembly for components with `RouteAttribute`. Each has `[Authorize]`, or relies on the fallback, except `AccessDenied`, which has `[AllowAnonymous]`.
  - AppHost `AdminReferencesExternalServicesByParameter`: resources `identity-api`, `businessrules-api`, and `catalog-api` are `ExternalServiceResource`. Parameters `identity-url`, `businessrules-url`, and `catalog-url` exist, and `eshopadmin` references all three.
  - E2E `e2e/login.setup.ts`: signs in through Identity.API as `PUBLISHER_USERNAME` and `EDITOR_USERNAME` and saves `playwright/.auth/publisher.json` and `editor.json`.
  - E2E `SignInTest.spec.ts` "publisher sees the admin home with their name" (role `banner` contains the user's display name) and "sign out returns to the sign-in page".
- [ ] **Step 2: Run** `dotnet test --solution eShopAdmin.slnx`. Expected: build FAIL (types missing).
- [ ] **Step 3: Implement.** Playwright config: projects `setup`, `publisher` (storage state `publisher.json`), and `editor` (`editor.json`). `baseURL` = `ADMIN_URL` (default `https://localhost:7250`). `webServer` runs `dotnet run --project src/eShopAdmin.AppHost` with `reuseExistingServer: !process.env.CI`. eShop must already be running (README).
- [ ] **Step 4: Run** `dotnet test --solution eShopAdmin.slnx` and `dotnet build eShopAdmin.slnx`. Expected: PASS, 0 warnings. With eShop running (`aspire run` in eShop), run `npx playwright test e2e/SignInTest.spec.ts`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): scaffold eShopAdmin with Aspire AppHost, sign-in and access policies`

### Task 4: eShopAdmin — BusinessRules.API client and re-authentication

**Files:**
- Create: `src/eShopAdmin/BusinessRules/{BusinessRulesClient,BusinessRulesDtos,SaveDraftResult,PublishResult,ProblemTypes,ReauthenticationRequiredException}.cs`
- Create: `src/eShopAdmin/Auth/{AccessTokenHandler,ReauthenticateMiddleware}.cs`
- Modify: `src/eShopAdmin/Program.cs` (typed client + middleware)
- Test: `tests/eShopAdmin.UnitTests/BusinessRules/BusinessRulesClientTests.cs`, `Auth/ReauthenticateMiddlewareTests.cs`

**Interfaces:**
- Consumes: BusinessRules.API v1 admin endpoints (`ListRuleTypes`, `ListRuleVersions`, `CreateRuleDraft`, `UpdateRuleDraft`, `PublishRuleVersion`, `ListRuleAudit`) and their problem types in `src/BusinessRules.API/Services/IPublishGuard.cs` (`https://eshop/problems/{concurrent-publish, v1-clients-active, publish-guard-unavailable, invalid-guard-override-reason}`).
- Produces:
  - DTO copies (namespace `eShopAdmin.BusinessRules`, enums deserialized as strings): `RuleTypeDto(string Id, string Name, bool AffectsPrice, RuleTargeting Targeting, JsonElement Default, JsonElement ParameterSchema)`, `RuleVersionDto(Guid Id, string RuleTypeId, string Scope, int VersionNumber, JsonElement Parameters, DateTimeOffset EffectiveFrom, DateTimeOffset? EffectiveTo, RuleVersionStatus Status, string CreatedBy, DateTimeOffset CreatedAt, string? PublishedBy, DateTimeOffset? PublishedAt)`, `AuditEntryDto(long Id, DateTimeOffset OccurredAt, string ActorId, AuditAction Action, string RuleTypeId, string Scope, Guid? RuleVersionId, string? BeforeJson, string? AfterJson, string? Reason)`, `CreateDraftRequest(string RuleTypeId, string Scope, JsonElement Parameters, DateTimeOffset EffectiveFrom, DateTimeOffset? EffectiveTo)`, `UpdateDraftRequest(JsonElement Parameters, DateTimeOffset EffectiveFrom, DateTimeOffset? EffectiveTo)`, `PaginatedItems<T>(int PageIndex, int PageSize, long Count, IReadOnlyList<T> Data)`. Enums: `RuleTargeting { Global, Targeted }`, `RuleVersionStatus { Draft, Published, Superseded }`, `AuditAction { DraftCreated, DraftUpdated, Published, GuardOverridden }`.
  - `public abstract record SaveDraftResult { Saved(RuleVersionDto Version); Invalid(IReadOnlyDictionary<string, string[]> Errors); Conflict(string Detail); NotFound; }`
  - `public abstract record PublishResult { Published(long SnapshotVersion); V1ClientsActive(string Detail); GuardUnavailable; InvalidOverrideReason(string Detail); ConcurrentPublish; Conflict(string Detail); NotFound; Forbidden; }`
  - `public sealed class BusinessRulesClient(HttpClient http, ILogger<BusinessRulesClient> logger)`. Base `https+http://businessrules-api`, `.AddApiVersion(1.0)`, `AccessTokenHandler`.
    - `Task<IReadOnlyList<RuleTypeDto>> GetRuleTypesAsync(CancellationToken ct)`
    - `Task<IReadOnlyList<RuleVersionDto>> GetVersionsAsync(string ruleTypeId, CancellationToken ct)` (scope `default`)
    - `Task<SaveDraftResult> CreateDraftAsync(CreateDraftRequest request, CancellationToken ct)`
    - `Task<SaveDraftResult> UpdateDraftAsync(Guid id, UpdateDraftRequest request, CancellationToken ct)`
    - `Task<PublishResult> PublishAsync(Guid id, string? guardOverrideReason, CancellationToken ct)`. The body is `{ "guardOverrideReason": … }`.
    - `Task<PaginatedItems<AuditEntryDto>> GetAuditAsync(int pageIndex, int pageSize, CancellationToken ct)`
    - Every method: 401 → `ReauthenticationRequiredException`. Any status not listed in the result types → `HttpRequestException`.
    - Logs `"Rule draft {RuleTypeId} saved"` and `"Rule version {RuleVersionId} published as snapshot {SnapshotVersion}"`, with no user data.
  - `AccessTokenHandler : DelegatingHandler(IHttpContextAccessor)`: adds `Authorization: Bearer` from `HttpContext.GetTokenAsync("access_token")`.
  - `ReauthenticateMiddleware`: catches `ReauthenticationRequiredException` and calls `ChallengeAsync(OpenIdConnectDefaults.AuthenticationScheme, new() { RedirectUri = PathBase + Path + QueryString })`. Registered after authentication, before Razor components.

- [ ] **Step 1: Write failing tests** (stub `HttpMessageHandler` that records the request and returns a canned response)
  - `PublishSendsApiVersionAndOverrideReason`: query has `api-version=1.0`, body `guardOverrideReason` = "Spring sale"
  - `PublishMapsV1ClientsActiveWithDetail`: 409 with type `…/v1-clients-active` and detail "3 calls" → `V1ClientsActive("3 calls")`
  - `PublishMapsGuardUnavailable` (503), `PublishMapsInvalidOverrideReason` (400 typed), `PublishMapsConcurrentPublish` (409 typed), `PublishMapsForbidden` (403)
  - `PublishOfNonDraftMapsToConflict`: 409 without a known type → `Conflict(detail)` (Review Focus 5)
  - `CreateDraftMapsValidationProblemErrors`: 400 `ValidationProblemDetails` → `Invalid` with the same dictionary
  - `UpdateDraftMapsConflictAndNotFound`
  - `UnauthorizedThrowsReauthenticationRequired` (Review Focus 1)
  - `GetVersionsParsesStringEnumsAndJsonParameters`: status `"Published"`, `parameters.minutes == 10`
  - `MiddlewareChallengesWithReturnUrl`: `DefaultHttpContext` with path `/rules/ORD-001` and query `?x=1`, next throws → an `IAuthenticationService` substitute receives a challenge with `RedirectUri == "/rules/ORD-001?x=1"`
- [ ] **Step 2: Run** `dotnet test --project tests/eShopAdmin.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): add BusinessRules.API client with publish guard results and re-authentication`

### Task 5: eShopAdmin — rule forms, JSON mapping, server error mapping

**Files:**
- Create: `src/eShopAdmin/Rules/{DraftInput,RuleForms,GracePeriodForm,ShortfallPolicyForm,ShippingFeeForm,PriceAdjustmentForm,PromotionForm,TargetForm,ValidationErrorMapper,ParameterSummary}.cs`
- Test: `tests/eShopAdmin.UnitTests/Rules/{RuleFormsTests,ValidationErrorMapperTests,ParameterSummaryTests}.cs`

**Interfaces:**
- Consumes: phase 2 parameter shapes (P2-D1; examples in phase 2 Task 2); `RuleVersionDto`, `RuleTypeDto` (Task 4).
- Produces (namespace `eShopAdmin.Rules`; mutable classes with parameterless constructors so `[SupplyParameterFromForm]` can bind them):
  - `public static class RuleForms { public static bool IsSupported(string ruleTypeId); }` for `ORD-001`, `ORD-013`, `SHP-001`, `PRC-001`, `PRM-001`
  - `GracePeriodForm { int Minutes }` ↔ `{"minutes":…}`
  - `ShortfallPolicyForm { string Policy }` (`"AllOrNothing"` | `"PartialFulfilment"`) ↔ `{"policy":…}`
  - `ShippingFeeForm { decimal Fee; decimal? FreeShippingThreshold }`
  - `TargetForm { string Kind; int? Id }` (`All` | `Brand` | `Type` | `Item`; `Id` null for `All`)
  - `PriceAdjustmentForm { List<PriceAdjustmentEntryForm> Entries }`, `PriceAdjustmentEntryForm { TargetForm Target; string Mode; decimal Value }`
  - `PromotionForm { List<LinePromotionForm> LinePromotions; List<OrderPromotionForm> OrderPromotions }`, `LinePromotionForm { TargetForm Target; string Kind; decimal Value; int? MinUnits; string Label }`, `OrderPromotionForm { string Kind; decimal Value; decimal MinSubtotal; string Label }`
  - `public sealed class DraftInput`
    - `DateTime EffectiveFromUtc; DateTime? EffectiveToUtc`, plus one property per form above (all non-null; only the one matching the rule type is used)
    - `static DraftInput? FromParameters(string ruleTypeId, JsonElement parameters, DateTimeOffset effectiveFrom, DateTimeOffset? effectiveTo)`: returns null for an unsupported type, or when the JSON has a member the form doesn't model, lacks a required one, or has a wrong kind (strict, so a save can't drop data)
    - `JsonElement ToParameters(string ruleTypeId)`: camelCase, string enums, nulls written as `null` (PRM-001 `minUnits`)
    - `void AddEntry(string list)`, `void RemoveEntry(string list, int index)` for `"Entries"`, `"LinePromotions"`, `"OrderPromotions"` (used by the named form handlers in Task 7)
  - `public static class ValidationErrorMapper { public static FieldErrors Map(string ruleTypeId, IReadOnlyDictionary<string, string[]> errors); }`, `public sealed record FieldErrors(IReadOnlyDictionary<string, string[]> ByField, IReadOnlyList<string> Summary)`
    - Messages under `Parameters` that start with a JSON path `^[a-z]\w*(\[\d+\])?(\.[a-z]\w*(\[\d+\])?)*: ` map to the form field `"{FormProperty}.{PascalCasePath}"`, e.g. `PRC-001` + `entries[2].value: …` → `PriceAdjustment.Entries[2].Value`. The message keeps only the text after `": "`.
    - Keys `EffectiveFrom` / `EffectiveTo` → `EffectiveFromUtc` / `EffectiveToUtc`.
    - Everything else goes to `Summary`.
  - `public static class ParameterSummary { public static string Describe(string ruleTypeId, JsonElement parameters); }`. Exact output: `ORD-001` → "10 minutes" ("1 minute" for 1); `ORD-013` → "All or nothing" / "Partial fulfilment"; `SHP-001` → "Fee $4.99, free from $50.00" or "Fee $4.99"; `PRC-001` → "3 price adjustments" ("1 price adjustment", "No price adjustments"); `PRM-001` → "2 line, 1 order promotions"; unknown or unmappable → "Custom value".

- [ ] **Step 1: Write failing tests**

```csharp
[TestMethod] public void PromotionExampleRoundTrips()
{   var json = J("""{"linePromotions":[{"target":{"kind":"Item","id":99},"kind":"PercentOff","value":10,"minUnits":null,"label":"10% off"}],"orderPromotions":[]}""");
    var input = DraftInput.FromParameters("PRM-001", json, Now, null)!;
    Assert.IsTrue(JsonElement.DeepEquals(json, input.ToParameters("PRM-001"))); }

[TestMethod] public void MapsJsonPathErrorsToFormFields()                    // Review Focus 2
{   var mapped = ValidationErrorMapper.Map("PRC-001", new Dictionary<string, string[]> {
        ["Parameters"] = ["entries[2].value: percent off must be between 0 and 90."], ["EffectiveTo"] = ["must be after EffectiveFrom"] });
    CollectionAssert.AreEqual(new[] { "percent off must be between 0 and 90." }, mapped.ByField["PriceAdjustment.Entries[2].Value"].ToArray());
    Assert.IsTrue(mapped.ByField.ContainsKey("EffectiveToUtc")); Assert.AreEqual(0, mapped.Summary.Count); }
```

  - `EachSupportedTypeRoundTripsItsDefault` (DataRows for the five built-in defaults: `{"minutes":1}`, `{"policy":"AllOrNothing"}`, `{"fee":0,"freeShippingThreshold":null}`, `{"entries":[]}`, `{"linePromotions":[],"orderPromotions":[]}`)
  - `PriceAdjustmentExampleRoundTrips` (phase 2 Task 2 example)
  - `FromParametersReturnsNullForUnrecognisedShape` (DataRows: `ORD-001` `{"minutes":5,"extra":1}`, `ORD-001` `{}`, `SHP-001` `{"fee":"x"}`, `XYZ-001` `{}`) (Review Focus 3)
  - `TargetAllWritesNullId`
  - `UnmappableErrorsGoToSummary` (`Parameters` message without a path; key `RuleTypeId`)
  - `AddAndRemoveEntryKeepOrder`
  - `ParameterSummaryDescribesEachType` (DataRows with the exact strings above)
- [ ] **Step 2: Run** `dotnet test --project tests/eShopAdmin.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): add typed rule forms with JSON and validation error mapping`

### Task 6: eShopAdmin — rules list and version history pages

**Files:**
- Create: `src/eShopAdmin/Rules/RuleStatusSummary.cs`
- Create: `src/eShopAdmin/Components/Pages/Rules/{RulesIndex.razor,RuleHistory.razor}`
- Modify: `src/eShopAdmin/Components/Pages/Home.razor` (redirect `/` → `/rules`)
- Test: `tests/eShopAdmin.UnitTests/Rules/RuleStatusSummaryTests.cs`; E2E `e2e/RulesListTest.spec.ts`

**Interfaces:**
- Consumes: `BusinessRulesClient.GetRuleTypesAsync/GetVersionsAsync` (Task 4); `ParameterSummary`, `RuleForms.IsSupported` (Task 5).
- Produces:
  - `public sealed record RuleStatusSummary(RuleVersionDto? Current, RuleVersionDto? NextScheduled, int DraftCount) { public static RuleStatusSummary From(IReadOnlyList<RuleVersionDto> versions, DateTimeOffset now); }`. `Current` follows phase 1 D5: published versions with `EffectiveFrom ≤ now < (EffectiveTo ?? ∞)`, the latest `EffectiveFrom` wins, and a tie goes to the highest `VersionNumber`; null means the built-in default applies. `NextScheduled` is the published version with the earliest `EffectiveFrom > now`.
  - `/rules` (`RulesIndex`): a `<table>` with a caption and one row per rule type, ordered by id. Columns: *Rule* (link "{Id} {Name}" to the history page), *Changes prices* (Yes/No), *Current value* (`ParameterSummary` or the default copy), *Scheduled* ("v{N} from {yyyy-MM-dd HH:mm} UTC" or "—"), *Drafts* (count). Unsupported types show the unknown-rule-type copy in the row.
  - `/rules/{RuleTypeId}` (`RuleHistory`): the rule name as `<h1>`, the status summary, and a *New draft* link (hidden for unsupported types). History table, newest first: *Version*, *Status*, *Value* (`ParameterSummary`), *Effective (UTC)*, *Created*, *Published*. Each draft row has *Edit*, plus *Publish* only when the user has the `Publish` policy. Each row has a `<details>` with the raw parameters JSON. An unknown id → 404 page.

- [ ] **Step 1: Write failing tests**
  - `CurrentIsLatestEffectivePublishedVersion` (v1 from −2 d, v2 from −1 d, v3 from +1 d → current v2, scheduled v3)
  - `ExpiredVersionIsNotCurrent` (v2 to −1 h → v1)
  - `DraftsAndSupersededAreNeverCurrent`
  - `NoPublishedVersionMeansDefault` (`Current == null`)
  - `TieOnEffectiveFromPicksHighestVersion`
  - E2E (`publisher` project) "lists the five rule types": the table has rows for `PRC-001`, `PRM-001`, `SHP-001`, `ORD-001`, and `ORD-013`. `ORD-001` *Changes prices* is "No"; `SHP-001` is "Yes".
  - E2E (`editor` project) "editor sees history without Publish": open `ORD-001` history; no element with role `link` or `button` and name "Publish" is present.
  - `UnknownRuleTypeIsListedReadOnly`: unit-level, a rule type id `XYZ-001` gives `RuleForms.IsSupported == false` and `ParameterSummary` "Custom value". The page uses those two calls only (Review Focus 3).
- [ ] **Step 2: Run** `dotnet test --project tests/eShopAdmin.UnitTests`, then (eShop running) `npx playwright test e2e/RulesListTest.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): list rules with current, scheduled and draft versions`

### Task 7: eShopAdmin — create and edit drafts (schedule)

**Files:**
- Create: `src/eShopAdmin/Catalog/{CatalogLookupClient,CatalogOption}.cs`
- Create: `src/eShopAdmin/Components/Pages/Rules/{DraftEditor.razor,Forms/GracePeriodFields.razor,Forms/ShortfallPolicyFields.razor,Forms/ShippingFeeFields.razor,Forms/PriceAdjustmentFields.razor,Forms/PromotionFields.razor,Forms/TargetFields.razor,Forms/EffectiveWindowFields.razor}`
- Modify: `src/eShopAdmin/Rules/DraftInput.cs` (`ValidateWindow`)
- Test: `tests/eShopAdmin.UnitTests/Rules/DraftWindowTests.cs`, `Catalog/CatalogLookupClientTests.cs`; E2E `e2e/EditDraftTest.spec.ts`

**Interfaces:**
- Consumes: `DraftInput`, `ValidationErrorMapper` (Task 5); `CreateDraftAsync`, `UpdateDraftAsync`, `GetVersionsAsync`, `GetRuleTypesAsync` (Task 4); Catalog.API `GET api/catalog/catalogtypes`, `GET api/catalog/catalogbrands`, `GET api/catalog/items/by?ids=` (v1.0).
- Produces:
  - `public sealed record CatalogOption(int Id, string Name)`
  - `public sealed class CatalogLookupClient(HttpClient http, ILogger<CatalogLookupClient> logger)`. Base `https+http://catalog-api`, `.AddApiVersion(1.0)`, no token.
    - `Task<IReadOnlyList<CatalogOption>?> GetTypesAsync(CancellationToken ct)` and `GetBrandsAsync`: null on any failure, with a warning log
    - `Task<IReadOnlyDictionary<int, string>> GetItemNamesAsync(IReadOnlyCollection<int> ids, CancellationToken ct)`: empty on failure
  - `IReadOnlyDictionary<string, string[]> DraftInput.ValidateWindow(DateTimeOffset now, out bool startsInPast)`: error on `EffectiveToUtc` when it is set and ≤ `EffectiveFromUtc`. `startsInPast` = `EffectiveFromUtc < now`. Both inputs are treated as `DateTimeKind.Utc`.
  - `CreateDraftRequest DraftInput.ToCreateRequest(string ruleTypeId)` (scope `default`) and `UpdateDraftRequest DraftInput.ToUpdateRequest(string ruleTypeId)`: parameters from `ToParameters`, times as UTC `DateTimeOffset`, `EffectiveTo` null when `EffectiveToUtc` is null.
  - Routes:
    - `/rules/{RuleTypeId}/drafts/new`: pre-filled from the current version (Task 6 `RuleStatusSummary.Current`). `?from=default` pre-fills from `RuleTypeDto.Default`. `EffectiveFromUtc` defaults to now, rounded up to the next minute.
    - `/rules/{RuleTypeId}/drafts/{Id:guid}`: loads the draft from `GetVersionsAsync` and returns 404 when it is missing. If the version isn't a draft, the page shows it read-only with a link to history.
  - One `EditForm` (`FormName="draft"`, enhanced) with submit buttons: *Save draft* (handler `save`), *Add entry* / *Add line promotion* / *Add order promotion* (handler `add-{list}`), *Remove* per row (handler `remove-{list}-{index}`), and *Start from built-in default* (a link to `?from=default`). Each input has a visible `<label>`. Field errors render with `ValidationMessage`, linked through `aria-describedby`. The summary renders in a `role="alert"` region at the top and gets focus after a failed save.
  - On `Saved` → redirect to the history page with the status message "Draft v{N} saved." `Invalid` → map with `ValidationErrorMapper`. `Conflict(detail)` → the other-409 copy.

- [ ] **Step 1: Write failing tests**
  - `WindowRejectsEndNotAfterStart` (end = start → error on `EffectiveToUtc`) (Review Focus 4)
  - `WindowFlagsPastStart` (start = now − 1 min → `startsInPast`, no error)
  - `EmptyEndMeansOpenEnded` (`ToCreateRequest` with `EffectiveToUtc == null` → `EffectiveTo == null`; `EffectiveFrom.Offset == TimeSpan.Zero`)
  - `CatalogTypesReturnNullWhenCatalogFails` (500 → null), `ItemNamesEmptyWhenCatalogFails`
  - E2E (`publisher`) "schedule a grace period draft": open `/rules/ORD-001`, choose *New draft*, fill "Minutes" with `61`, and save. The error is shown next to "Minutes" (from the server). Change it to `10`, set "Starts (UTC)" to tomorrow 09:00, and save. The history shows a *Draft* row with value "10 minutes" and the effective date.
  - E2E (`publisher`) "price adjustment rows can be added and removed": on `PRC-001`, *Add entry* twice and *Remove* the first. One row remains, and its "Target type" select lists catalog types by name.
- [ ] **Step 2: Run** unit tests, then `npx playwright test e2e/EditDraftTest.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): create and schedule rule drafts with typed forms`

### Task 8: eShopAdmin — publish with guard override

**Files:**
- Create: `src/eShopAdmin/Components/Pages/Rules/PublishVersion.razor`
- Test: E2E `e2e/PublishRuleTest.spec.ts`

**Interfaces:**
- Consumes: `PublishAsync` and `PublishResult` (Task 4); `GetVersionsAsync`, `GetRuleTypesAsync`, `ParameterSummary`, `RuleStatusSummary` (Tasks 4–6).
- Produces:
  - `/rules/{RuleTypeId}/versions/{Id:guid}/publish`, `[Authorize(Policy = AdminPolicies.Publish)]`. An editor is redirected to `/access-denied` by the cookie handler.
  - The page shows `<h1>` "Publish {Name} v{N}", a two-column comparison (*Current* = `RuleStatusSummary.Current` summary or the default copy; *New* = this version's summary), the effective window, the past-start notice when it applies, and the price warning when `AffectsPrice`. The form (`FormName="publish"`) has a *Publish* button.
  - Outcomes:
    - `Published` → redirect to history with the published copy
    - `V1ClientsActive` → guard copy + detail, a required `<textarea>` (`maxlength=500`) with the guard reason label, and a *Publish anyway* button. The next submit sends the reason.
    - `InvalidOverrideReason` → field error on the reason
    - `GuardUnavailable` → guard-unavailable copy
    - `ConcurrentPublish` → concurrent copy
    - `Conflict` → other-409 copy
    - `NotFound` → 404
    - `Forbidden` → `/access-denied`
  - The reason is trimmed, and an empty reason is never sent.

- [ ] **Step 1: Write failing E2E tests**
  - (`publisher`) "publish a scheduled grace period": create an `ORD-001` draft of 10 minutes starting now (Task 7 flow), open *Publish*, and see the *Current*/*New* comparison. Click *Publish*. The history shows the version as *Published*, and the status message matches `/^Published\. Rules snapshot \d+\./`. `afterAll`: publish a draft made with *Start from built-in default* (P3-D10).
  - (`publisher`) "double submit shows a clear conflict": after publishing, go back and submit the same form again. The page shows text starting "This version can't be changed any more" (Review Focus 5).
  - (`editor`) "editor cannot publish": open the publish URL of an existing draft directly. The access-denied copy is visible (spec AC: RulesEditor can't publish; the API 403 is covered by BusinessRules.FunctionalTests `EditorCannotPublish`).
- [ ] **Step 2: Run** `npx playwright test e2e/PublishRuleTest.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): publish rule versions with the v1 clients guard override`

### Task 9: eShopAdmin — audit log

**Files:**
- Create: `src/eShopAdmin/Components/Pages/Audit/AuditLog.razor`, `src/eShopAdmin/Rules/AuditActionText.cs`
- Test: `tests/eShopAdmin.UnitTests/Rules/AuditActionTextTests.cs`; E2E `e2e/AuditLogTest.spec.ts`

**Interfaces:**
- Consumes: `GetAuditAsync` (Task 4).
- Produces:
  - `public static class AuditActionText { public static string Describe(AuditAction action); }` → "Draft created", "Draft updated", "Published", "Guard overridden"
  - `/audit?page={n}` (0-based, default 0, page size 20): a table with caption "Rule changes, newest first". Columns: *When (UTC)*, *Action*, *Rule*, *Version id*, *By*, *Reason*. *By* shows the actor id, plus " (you)" when it equals the signed-in user's `sub`. Each row has a `<details>` with the before and after JSON. Previous/next links are `<nav aria-label="Audit pages">` and are disabled at the ends.
- [ ] **Step 1: Write failing tests**
  - `DescribesEveryAuditAction` (DataRows for all four)
  - E2E (`publisher`) "audit shows the publish by me": after `PublishRuleTest` (or its own draft + publish of `ORD-001` default), the first rows include "Published" for `ORD-001` with "(you)".
- [ ] **Step 2: Run** unit tests and `npx playwright test e2e/AuditLogTest.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(admin): show the rules audit log`

### Task 10: eShopAdmin — CI workflows

**Files:**
- Create: `.github/workflows/pr-validation.yml` (`dotnet build eShopAdmin.slnx` + `dotnet test --solution eShopAdmin.slnx`; ignores `**/*.md`)
- Create: `.github/workflows/markdownlint.yml` (copy eShop's)
- Create: `.github/workflows/playwright.yml`
- Modify: `README.md` (how CI runs both AppHosts)

**Interfaces:**
- Consumes: repository variable `ESHOP_REPOSITORY`; eShop parameter `eshopadmin-url` (Task 2); eShopAdmin parameters (Task 3).
- Produces: `playwright.yml` on PR and push to `main`:
  1. Check out eShopAdmin, and `${{ vars.ESHOP_REPOSITORY }}` at `main` into `eshop/`.
  2. Set up .NET from `global.json` and Node LTS; run `npm ci` and `npx playwright install --with-deps chromium`.
  3. Start eShop in the background: `dotnet run --project eshop/src/eShop.AppHost` with `ESHOP_USE_HTTP_ENDPOINTS=1` and `Parameters__eshopadmin-url=http://localhost:5250`. Wait (5 min timeout) for `http://localhost:5223/.well-known/openid-configuration` and `http://localhost:5230/health`.
  4. `npm run test:e2e` with `ESHOP_USE_HTTP_ENDPOINTS=1`, `Parameters__identity-url=http://localhost:5223`, `ADMIN_URL=http://localhost:5250`, `PUBLISHER_USERNAME=alice`, `EDITOR_USERNAME=bob`, `PASSWORD=Pass123$` (seeded demo credentials, already public in eShop's workflow).
  5. Upload `playwright-report/` (30 days).

- [ ] **Step 1:** Write the three workflows.
- [ ] **Step 2:** Push the branch and open a draft PR, after Q Brown confirms pushing to the new remote.
- [ ] **Step 3:** Expected: `pr-validation`, `markdownlint`, and `playwright` are green on the draft PR. If `playwright` fails, read the uploaded report before changing anything (`systematic-debugging`).
- [ ] **Step 4: Commit** `ci(admin): add build, markdown and Playwright workflows`

### Task 11: Journey and architecture docs (eShopOrgMeta)

**Files (eShopOrgMeta):**
- `architecture/personas-and-journeys.md`: P5 drops "(planned)", auth = "Identity.API `eshopadmin` (PKCE), scope `businessrules.admin`, roles `RulesEditor`/`RulesPublisher`". J8 drops "(planned)", steps "Rules list → history → new draft (typed form, server validation) → schedule (UTC) → publish (guard override if prompted) → audit log", pages `/rules`, `/rules/{id}`, `/rules/{id}/drafts/…`, `/rules/{id}/versions/{v}/publish`, `/audit`, coverage **E2E** (eShopAdmin `e2e/*.spec.ts`).
- `architecture/business-rules-catalog.md` line 87: "editable through the API (Scalar) until eShopAdmin ships in phase 3" → "editable in eShopAdmin (`/rules`) or through the API (Scalar)".
- `standards/testing.md`: add an E2E row for eShopAdmin (`eShopAdmin/e2e`, needs eShop running) and the "no component-test library" rule (P3-D9).
- `standards/ci-cd.md`: add eShopAdmin's three workflows.
- `ways-of-working/onboarding.md`: running eShopAdmin next to eShop.
- `decisions/README.md` and ADR-0009: status → `Accepted` once its PR merges.
- Spec front matter: add the phase 3 PRs (eShop + eShopAdmin) to `implemented_in` once merged.

- [ ] **Step 1:** Make the edits. Cite eShopAdmin file paths as sources and update `last_reviewed`.
- [ ] **Step 2:** Check every relative link in the touched files resolves. Expected: no broken links.
- [ ] **Step 3: Commit** `docs(architecture): make J8 real with eShopAdmin`

### Phase 3 exit checks

- [ ] eShop: `dotnet build eShop.Web.slnf` 0 warnings; `dotnet test --solution eShop.Web.slnf` green; `npm run test:e2e` green (J1–J3, J5 and the phase 2 journeys).
- [ ] eShopAdmin: `dotnet build eShopAdmin.slnx` 0 warnings; `dotnet test --solution eShopAdmin.slnx` green; `npm run test:e2e` green locally and in CI.
- [ ] Manual WCAG 2.2 AA pass on `/rules`, a history page, the PRM-001 draft editor, publish, and `/audit`: keyboard-only through *new draft → save → publish*; visible focus; skip link works; every input labelled; errors announced (`role="alert"`) and linked to fields; text contrast ≥ 4.5:1; reflow at 320 CSS px without horizontal scrolling; unique page titles. Record the result in the PR description.
- [ ] Manual check: as `alice`, publish PRC-001 10 % off one catalog type from eShopAdmin. Within 60 s the storefront shows the reduced price (spec AC *T-shirt*, now through the UI). Then publish the default again from *Start from built-in default*.
- [ ] Onboarding checklist complete ([repo-onboarding-checklist.md](../../templates/repo-onboarding-checklist.md)). Q Brown turns on branch protection for `main` (PR + required checks `pr-validation`, `playwright`, `markdownlint`).
- [ ] ADR-0009 Accepted.
- [ ] `superpowers:requesting-code-review` → `finishing-a-development-branch`, once per repo. Each PR links the spec, this plan, and ADR-0009.

## Self-review notes

- **Spec coverage (phase 3 rollout item → task):**
  - Onboard repo (registry, workspace, specs/plans folders, baseline, CI, CODEOWNERS, AGENTS.md) → T1, T3, T10
  - Rules UI list / edit with validation / schedule / publish / history → T5–T8; audit → T9
  - Security (PKCE client, `businessrules.admin`, roles, server-side enforcement, no customer scopes) → T2, T3, T8
  - WCAG 2.2 AA → T3 layout, T7 form rules, exit checklist
  - Docs: personas/journeys J8, service catalog, system context, environments → T1, T11
- **Acceptance criteria touched here:** RulesEditor can't publish (UI → T8; API already in phase 1); T-shirt 10 % via UI (exit manual check); audit shows who published (T9).
- **Out of scope:** experiments UI (phase 4), results dashboards (phase 5), draft discard (P3-D11), deployment (P3-D2).
- **Type names are consistent across tasks:** `AdminPolicies`, `SessionLifetime`, `BusinessRulesClient`, `SaveDraftResult`, `PublishResult`, `ReauthenticationRequiredException`, `ReauthenticateMiddleware`, `DraftInput`, `RuleForms`, `ValidationErrorMapper`/`FieldErrors`, `ParameterSummary`, `RuleStatusSummary`, `CatalogLookupClient`/`CatalogOption`, `AuditActionText`.
