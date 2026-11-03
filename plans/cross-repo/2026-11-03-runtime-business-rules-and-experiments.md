---
title: "Runtime business rules and A/B experiments — implementation plan"
repo: "cross-repo"
status: Approved
approved_by: ["Q Brown (2026-11-03)"]
owner: "Q Brown"
created: 2026-11-03
last_reviewed: 2026-11-03
spec: "specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md"
scope: "Phase 1 (Foundation) in full; phases 2–5 outlined, each to be planned in its own file before it starts"
---

# Runtime Business Rules — Phase 1 (Foundation) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the server the pricing authority. Stand up BusinessRules.API so it can store, publish,
and audit rule versions and serve a rules snapshot. Move WebApp and ClientApp to server quotes plus
create order v2. No visible behaviour changes until a rule is published.

**Architecture:** A new T3 service, `BusinessRules.API` (Minimal API, Postgres `rulesdb`, outbox),
publishes immutable, versioned **rules snapshots**. A new shared library, `eShop.BusinessRules.Client`,
caches the snapshot in each consumer. It polls every 60 s and also refreshes on
`BusinessRulesSnapshotPublishedIntegrationEvent`, and it holds the pure **pricing calculator**.
Ordering.API (T1) fetches base prices from Catalog.API and prices locally from the cached snapshot.
It never calls BusinessRules.API on the request path.

**Tech Stack:** .NET 10, Aspire 13.6, Minimal APIs + Asp.Versioning, EF Core + Npgsql, RabbitMQ +
`IntegrationEventLogEF`, Duende IdentityServer 7.3, Blazor (WebApp + WebAppComponents), .NET MAUI
(ClientApp), YARP (mobile-bff), MSTest + NSubstitute, Playwright.

**Spec:** [specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md](../../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md)

## Prerequisites (Definition of Ready)

- [x] Spec `status` is `Approved` ([definition-of-ready-and-done.md](../../ways-of-working/definition-of-ready-and-done.md)). Approved by Q Brown, 2026-11-03.
- [x] Reviewers have accepted the **Plan decisions** below. They resolve items the spec leaves to the
      builder, plus three gaps the spec doesn't cover.
- [ ] Worktree/branch `feature/runtime-business-rules-and-experiments` in `eShop` (and
      `docs/runtime-business-rules-phase-1` in eShopOrgMeta), created with `using-git-worktrees`
      ([branching-and-commits.md](../../ways-of-working/branching-and-commits.md)).
- Open legal question (tracking consent) does **not** block phase 1; tracking starts in phase 5.

## Global Constraints

- Packages only via `eShop/Directory.Packages.props`; **no new third-party package** (spec *Constraints*; [tech-radar.md](../../standards/tech-radar.md)). Everything needed is already in CPM.
- New projects: `net10.0`, `<Nullable>enable</Nullable>`, warnings as errors (repo-wide).
- New test projects use **MSTest.Sdk + NSubstitute**. xUnit v3 is **Hold**, but existing xUnit projects (`Ordering.FunctionalTests`) are extended in place, not migrated.
- Minimal APIs in `Apis/<Name>Api.cs`; `NewVersionedApi()` + `HasApiVersion`; `TypedResults`; every endpoint has `.WithName/.WithSummary/.WithDescription/.WithTags`; errors are RFC 7807 ProblemDetails ([api-design.md](../../standards/api-design.md)).
- Publishing only through the outbox; handlers idempotent ([architecture-patterns.md](../../standards/architecture-patterns.md)).
- Ordering.API, WebApp, and mobile-bff (T1) **never** `WaitFor` BusinessRules.API (T3), and never call it synchronously while pricing or creating orders (spec *Resilience*).
- Rule *types* are code; rule *values* are data. Phase 1 ships **no production rule types**; PRC-001, PRM-001, SHP-001, ORD-001, and ORD-013 arrive in phase 2.
- Built-in defaults = today's behaviour: price = `CatalogItem.Price`, discount 0, shipping fee 0.
- Rounding: 2 decimals, `MidpointRounding.AwayFromZero`, per line (spec *Pricing*).
- Invariants: final unit price ≥ 0.01; `OrderItem.Discount` ≤ line total (ORD-010); order total ≥ 0.
- `Order.GetTotal()` = `Σ(units × unitPrice − discount) + shippingFee`.
- Snapshot poll interval **60 s**. On a cold start with no snapshot, use defaults (spec *Resilience*).
- No PII in logs, traces, or metrics; trace/log scopes may carry `order.id` and `rules.snapshot.version` only ([security.md](../../standards/security.md)).
- Admin scope `businessrules.admin` + roles `RulesEditor` / `RulesPublisher`; runtime scope `businessrules`; customer scopes are never reused.
- Every new entity states its retention tier ([data-retention.md](../../standards/data-retention.md)). Phase 1 entities and tiers are listed in Task 6, Task 9, and Task 11.
- Commits: Conventional Commits with service scope, e.g. `feat(businessrules): …`.

## Plan decisions (review before execution)

These go beyond the spec's text. Each one resolves a "left to the builder" item or a gap.

| # | Decision | Why |
|---|----------|-----|
| D1 | Phase 1 registers **no** production rule types. Tests use a `TestRuleType`. | Spec rollout puts rule types in phase 2. This way phase 1 can't publish a rule that no consumer evaluates. |
| D2 | **Service-to-service auth = OAuth client credentials** via Identity.API. Clients: `businessrules-reader` (scope `businessrules`, used by Ordering.API) and `businessrules-api` (scope `ordering.usage`). Secrets are Aspire `AddParameter(..., secret: true)` with generated, persisted defaults. Tokens come from a small `ServiceTokenHandler` (plain `HttpClient` POST to `/connect/token`, no new package). | Spec leaves the mechanism to the builder "within security.md". Identity.API stays the only issuer. Works locally and on ACA. |
| D3 | **Publish guard data source:** Ordering.API counts v1 create-order calls per UTC day in a new table `ordering.apiusagedaily`, and exposes `GET api/usage/create-order-v1?days=7` (scope `ordering.usage`). BusinessRules.API reads it when publishing a price-affecting rule and **fails closed** (503) if it can't. | The spec says the guard uses "the `ordering.create_order.v1.calls` metric over 7 days". BusinessRules.API can't read OTel metrics, and cross-DB reads are banned. The OTel counter is also emitted for dashboards and alerts. |
| D4 | Snapshot is serialized as JSON (System.Text.Json); ETag = `"<version>"`; consumers **don't** persist it across restarts (startup fetch with 5 s timeout, then defaults). | Spec leaves format and persistence to the builder. A stale price shown after a restart is caught by the v2 409 check. |
| D5 | **Effective-version resolution:** candidates = published versions with `EffectiveFrom ≤ now < (EffectiveTo ?? ∞)`. Pick the max `EffectiveFrom`; if tied, the max `VersionNumber`. If there are none, use the built-in default. On publish, `EffectiveFrom` earlier than now is clamped to now. A published version becomes `Superseded` when a newer, open-ended version with a later-or-equal `EffectiveFrom` is already effective. | The spec defines statuses but not overlap or back-dating rules. Clamping keeps the audit truthful. |
| D6 | Create order **v2 maps the full orders API** (all v1 routes plus the new POST), so clients switch their whole ordering client to `api-version=2.0`. | Avoids clients mixing versions per call. mobile-bff gets `2.0` added to its orders route. |
| D7 | If a WebApp catalog or item page can't get a quote, it falls back to `CatalogItem.Price` and logs a warning. Cart and checkout **require** a quote. | Page rendering must not fail (T1). The v2 409 still guarantees the charged total was shown. |
| D8 | Dev role seeding: `alice` = `RulesPublisher`, `bob` = `RulesEditor` (existing demo users, credentials already suppressed). Plus a `businessrulesswaggerui` client for Scalar. | Lets phases 1–2 use the admin API before eShopAdmin exists (spec rollout: "editable via API (Scalar) until phase 3"). |
| D9 | `eShop.BusinessRules.Client` also hosts `ServiceTokenHandler`, and WebApp references the library for quote DTOs. ClientApp (MAUI) keeps its own DTOs. | The library depends on hosting and EventBus, which don't fit MAUI. |

## Review Focus

Spec-implied inputs that no task's main tests cover, most likely to bite first. Each one has a test in
the task named.

1. **Ordering.API starts while BusinessRules.API is down or has never published.** Quotes and orders must still succeed at catalog prices. → Task 12, `QuoteSucceedsWithDefaultsWhenSnapshotSourceUnavailable`.
2. **A basket line for a product Catalog no longer returns (deleted item), or units ≤ 0.** Expect 400 ProblemDetails listing the ids, not 500, and no order created. → Task 2 and Task 13, `CreateOrderV2WithUnknownProductReturnsBadRequest`.
3. **Duplicate product ids in one request** (two lines for product 7). Merge to one line with summed units (ORD-009) before pricing, so `expectedTotal` matches. → Task 2, `CalculateMergesDuplicateProductLines`.
4. **Two publishers publish at the same moment.** Expect exactly one new snapshot version per publish, no duplicate version numbers, and the loser gets 409. → Task 8, `ConcurrentPublishProducesDistinctSnapshotVersions`.
5. **Replayed `x-requestid` on v2 after a 409.** A retry with the same request id and a corrected `expectedTotal` must still create the order. The idempotency record must only be written on success. → Task 13, `RetryAfterPriceConflictWithSameRequestIdCreatesOrder`.

---

## File map (eShop unless noted)

| Path | Responsibility |
|------|----------------|
| `src/eShop.BusinessRules.Client/` (new) | Rule type contract + registry, snapshot model + resolution, pricing calculator + quote model, snapshot cache/refresh/event handler, `ServiceTokenHandler`, DI extensions |
| `src/BusinessRules.API/` (new) | Admin + runtime endpoints, `RulesContext` (`rulesdb`), publishing, audit, guard, outbox |
| `tests/BusinessRules.UnitTests/` (new, MSTest) | Client lib and BusinessRules.API domain unit tests |
| `tests/BusinessRules.FunctionalTests/` (new, MSTest) | BusinessRules.API endpoint, auth, persistence tests |
| `src/Ordering.Domain/AggregatesModel/OrderAggregate/{Order,OrderItem}.cs` | Pricing fields, `ApplyPricing`, `GetTotal` fix |
| `src/Ordering.Infrastructure/` | EF config, migration + backfill, `ApiUsageDaily` |
| `src/Ordering.API/` | Catalog price client, `OrderPricingService`, `PricingApi` (quote), v2 create, v1 server pricing, usage endpoint, metrics |
| `src/Identity.API/` | Scopes, clients, roles, role claims |
| `src/eShop.AppHost/{AppHost,Extensions}.cs` | `rulesdb`, `businessrules-api`, references, secrets, mobile-bff routes |
| `src/eShop.ServiceDefaults/Extensions.cs` | Register meters `eShop.BusinessRules`, `eShop.BusinessRules.Client`, `eShop.Ordering` |
| `src/WebAppComponents/`, `src/WebApp/` | Display-price provider, quote-based cart/checkout, v2 + 409 prompt |
| `src/ClientApp/` | Quote + v2 + prices-changed dialog |
| eShopOrgMeta `decisions/0008-…`, `architecture/*`, `standards/data-retention.md` | ADR and docs |

---

### Task 1: ADR-0008 (eShopOrgMeta)

**Files:**
- Create: `eShopOrgMeta/decisions/0008-runtime-business-rules-service-and-in-house-experimentation.md` (from [templates/adr.md](../../templates/adr.md))
- Modify: `eShopOrgMeta/decisions/README.md` (index row), spec front matter `adrs:` (link the file)

**Interfaces:** Produces the ADR number that later docs cite.

- [ ] **Step 1:** Write the ADR. Context = gaps G-01 and G-03. Options = spec *Approaches considered* A/B/C. Decision = option A plus plan decisions D2 (client credentials) and D3 (guard data source). Consequences = new T3 service, T1 independence rule, follow-ups (phase 2–5 docs). Status `Proposed`, Scope `org`, Related = spec + this plan.
- [ ] **Step 2:** Add the index row `| [0008](0008-…md) | Runtime business rules service and in-house experimentation | Proposed |`.
- [ ] **Step 3:** Open the eShopOrgMeta PR. The status moves to `Accepted` when merged. Phase 1 can't be marked Done until it is Accepted (spec rollout).
- [ ] **Step 4: Commit** `docs(adr): propose ADR-0008 runtime business rules service`

### Task 2: Shared library core — rule types, snapshot, pricing calculator

**Files:**
- Create: `src/eShop.BusinessRules.Client/eShop.BusinessRules.Client.csproj` (`Microsoft.NET.Sdk`, `net10.0`, Nullable; refs `EventBus`, `eShop.ServiceDefaults`)
- Create: `src/eShop.BusinessRules.Client/Rules/{IRuleType,RuleTargeting,RuleTypeRegistry,RuleScopes}.cs`
- Create: `src/eShop.BusinessRules.Client/Snapshot/{RulesSnapshot,SnapshotRuleVersion}.cs`
- Create: `src/eShop.BusinessRules.Client/Pricing/{QuoteLine,CatalogPrice,PricedLine,PriceQuote,PricingCalculator,UnknownProductsException,PriceRounding}.cs`
- Create: `tests/BusinessRules.UnitTests/BusinessRules.UnitTests.csproj` (MSTest.Sdk, NSubstitute), `TestRuleType.cs`
- Test: `tests/BusinessRules.UnitTests/{RulesSnapshotTests,PricingCalculatorTests,RuleTypeRegistryTests}.cs`
- Modify: `eShop.slnx`, `eShop.Web.slnf` (add both projects)

**Interfaces:**
- Produces:
  - `enum RuleTargeting { Global, Targeted }`; `static class RuleScopes { public const string Default = "default"; }`
  - `interface IRuleType { string Id { get; } string Name { get; } bool AffectsPrice { get; } RuleTargeting Targeting { get; } JsonElement Default { get; } string ParameterSchemaJson { get; } IReadOnlyList<string> Validate(JsonElement parameters); }`
  - `sealed class RuleTypeRegistry(IEnumerable<IRuleType> types) { IReadOnlyCollection<IRuleType> All { get; } IRuleType? Find(string id); }`; `IServiceCollection AddRuleType<T>() where T : class, IRuleType`
  - `sealed record SnapshotRuleVersion(string RuleTypeId, string Scope, int VersionNumber, DateTimeOffset EffectiveFrom, DateTimeOffset? EffectiveTo, JsonElement Parameters)`
  - `sealed record RulesSnapshot(long Version, DateTimeOffset PublishedAt, IReadOnlyList<SnapshotRuleVersion> Rules) { static RulesSnapshot Empty; SnapshotRuleVersion? FindEffective(string ruleTypeId, string scope, DateTimeOffset now); }`
  - `sealed record QuoteLine(int ProductId, int Units)`
  - `sealed record CatalogPrice(int ProductId, string ProductName, decimal BasePrice, int CatalogTypeId, int CatalogBrandId, string? PictureUrl)`
  - `sealed record PricedLine(int ProductId, string ProductName, string? PictureUrl, int Units, decimal BasePrice, decimal UnitPrice, decimal Discount, string? PromotionLabel) { decimal LineTotal => Units * UnitPrice - Discount; }`
  - `sealed record PriceQuote(IReadOnlyList<PricedLine> Lines, decimal Subtotal, decimal DiscountTotal, decimal ShippingFee, decimal Total, long SnapshotVersion)`. `Subtotal = Σ Units×UnitPrice`; `DiscountTotal = Σ Discount`; `Total = Subtotal − DiscountTotal + ShippingFee`.
  - `sealed class PricingCalculator { PriceQuote Calculate(IReadOnlyList<QuoteLine> lines, IReadOnlyDictionary<int, CatalogPrice> catalog, RulesSnapshot snapshot, DateTimeOffset now); }`. Throws `UnknownProductsException(IReadOnlyList<int> ProductIds)` and `ArgumentException` when units ≤ 0 or there are no lines.
  - `static class PriceRounding { decimal Round(decimal value) => Math.Round(value, 2, MidpointRounding.AwayFromZero); }`

- [ ] **Step 1: Write failing tests**

```csharp
[TestMethod] public void FindEffectiveReturnsNullWhenNoVersions()
    => Assert.IsNull(RulesSnapshot.Empty.FindEffective("TST-001", RuleScopes.Default, Now));

[TestMethod] public void FindEffectivePicksLatestEffectiveFromNotInFuture()
{   // v1 from Now-2d, v2 from Now-1d, v3 from Now+1d  → v2
    Assert.AreEqual(2, Snapshot(V(1, -2), V(2, -1), V(3, +1)).FindEffective("TST-001", "default", Now)!.VersionNumber); }

[TestMethod] public void FindEffectiveSkipsExpiredVersionAndFallsBackToEarlier()
{   // v1 from -3d open-ended; v2 from -2d to -1d → v1
    Assert.AreEqual(1, Snapshot(V(1, -3), V(2, -2, to: -1)).FindEffective("TST-001", "default", Now)!.VersionNumber); }

[TestMethod] public void FindEffectiveBreaksEffectiveFromTieByHighestVersion() { /* v1,v2 same from → 2 */ }

[TestMethod] public void CalculateWithEmptySnapshotChargesCatalogPrice()
{
    var q = Calc([new(1, 2), new(2, 1)], Catalog((1, 19.50m), (2, 8.00m)));
    Assert.AreEqual(47.00m, q.Subtotal); Assert.AreEqual(0m, q.DiscountTotal);
    Assert.AreEqual(0m, q.ShippingFee); Assert.AreEqual(47.00m, q.Total); Assert.AreEqual(0L, q.SnapshotVersion);
    Assert.AreEqual(19.50m, q.Lines[0].BasePrice); Assert.AreEqual(19.50m, q.Lines[0].UnitPrice);
}

[TestMethod] public void CalculateMergesDuplicateProductLines()            // Review Focus 3
{   var q = Calc([new(7, 1), new(7, 2)], Catalog((7, 10m)));
    Assert.AreEqual(1, q.Lines.Count); Assert.AreEqual(3, q.Lines[0].Units); Assert.AreEqual(30m, q.Total); }

[TestMethod] public void CalculateThrowsForUnknownProducts()                // Review Focus 2
{   var ex = Assert.ThrowsExactly<UnknownProductsException>(() => Calc([new(1, 1), new(99, 1)], Catalog((1, 5m))));
    CollectionAssert.AreEqual(new[] { 99 }, ex.ProductIds.ToArray()); }

[TestMethod] [DataRow(0)] [DataRow(-1)]
public void CalculateRejectsNonPositiveUnits(int units)
    => Assert.ThrowsExactly<ArgumentException>(() => Calc([new(1, units)], Catalog((1, 5m))));

[TestMethod] [DataRow(1.005, 1.01)] [DataRow(2.345, 2.35)] [DataRow(-0.005, -0.01)]
public void RoundIsHalfAwayFromZero(double input, double expected)
    => Assert.AreEqual((decimal)expected, PriceRounding.Round((decimal)input));

[TestMethod] public void RegistryFindReturnsRegisteredTypeAndNullOtherwise() { /* TestRuleType "TST-001" */ }
```

- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests`. Expected: build FAIL (types missing).
- [ ] **Step 3: Implement** the types above. `FindEffective` follows plan decision D5. In phase 1, `Calculate` = merge lines by `ProductId` (order of first appearance), check unknown ids (all of them, sorted), build `PricedLine` with `UnitPrice = BasePrice`, `Discount = 0`, shipping 0, and `SnapshotVersion = snapshot.Version`. Keep the evaluation order as named private steps (`ApplyPriceAdjustments`, `ApplyLinePromotions`, `ApplyOrderPromotion`, `ApplyShipping`) that are identity functions for now, so phase 2 can fill them in without restructuring.
- [ ] **Step 4: Run** the same command. Expected: all PASS.
- [ ] **Step 5: Commit** `feat(businessrules): add rules snapshot model and pricing calculator`

### Task 3: Shared library runtime — snapshot cache, refresh, event, service token

**Files:**
- Create: `src/eShop.BusinessRules.Client/Snapshot/{IRulesSnapshotSource,HttpRulesSnapshotSource,RulesSnapshotCache,RulesSnapshotRefreshService,BusinessRulesClientOptions,BusinessRulesClientMetrics}.cs`
- Create: `src/eShop.BusinessRules.Client/IntegrationEvents/{BusinessRulesSnapshotPublishedIntegrationEvent,BusinessRulesSnapshotPublishedIntegrationEventHandler}.cs`
- Create: `src/eShop.BusinessRules.Client/Auth/{ServiceTokenHandler,ServiceClientOptions}.cs`
- Create: `src/eShop.BusinessRules.Client/BusinessRulesClientExtensions.cs`
- Modify: `src/eShop.ServiceDefaults/Extensions.cs` (≈ line 64): `.AddMeter("eShop.BusinessRules.Client")`, `.AddMeter("eShop.BusinessRules")`, `.AddMeter("eShop.Ordering")`
- Test: `tests/BusinessRules.UnitTests/{RulesSnapshotCacheTests,HttpRulesSnapshotSourceTests,ServiceTokenHandlerTests}.cs`

**Interfaces:**
- Consumes: `RulesSnapshot` (Task 2).
- Produces:
  - `interface IRulesSnapshotSource { Task<SnapshotFetchResult> FetchAsync(long? knownVersion, CancellationToken ct); }`; `sealed record SnapshotFetchResult(bool NotModified, RulesSnapshot? Snapshot)`
  - `HttpRulesSnapshotSource(HttpClient)`: `GET api/rules/snapshot?api-version=1.0`, sends `If-None-Match: "<knownVersion>"`, 304 → `NotModified`
  - `sealed class RulesSnapshotCache(IRulesSnapshotSource, TimeProvider, ILogger<RulesSnapshotCache>) { RulesSnapshot Current { get; } DateTimeOffset? LastRefreshedAt { get; } Task<bool> RefreshAsync(CancellationToken ct); }`. Starts at `RulesSnapshot.Empty`, never throws, never moves the version backwards.
  - `RulesSnapshotRefreshService : BackgroundService`: initial `RefreshAsync` with a 5 s timeout, then `PeriodicTimer(options.PollInterval)`
  - `record BusinessRulesSnapshotPublishedIntegrationEvent(long SnapshotVersion, DateTimeOffset PublishedAt) : IntegrationEvent`; handler calls `cache.RefreshAsync` only when `SnapshotVersion > cache.Current.Version` (idempotent)
  - `sealed class BusinessRulesClientOptions { TimeSpan PollInterval = TimeSpan.FromSeconds(60); }`
  - Meter `eShop.BusinessRules.Client`: observable gauges `rules.snapshot.version`, `rules.snapshot.age_seconds`
  - `ServiceTokenHandler : DelegatingHandler` (options `ServiceClient:ClientId`, `ServiceClient:ClientSecret`, `Identity:Url`) posts `grant_type=client_credentials&scope=<scope>` to `{Identity:Url}/connect/token` and caches the token until `expires_in − 60 s`
  - `IHttpClientBuilder AddServiceToken(this IHttpClientBuilder builder, string scope)`
  - `IHostApplicationBuilder AddBusinessRulesClient(this IHostApplicationBuilder builder)`: registers `TimeProvider.System` (TryAdd), the `RuleTypeRegistry`, `PricingCalculator`, cache, refresh service, and the typed `HttpRulesSnapshotSource` with base `https+http://businessrules-api` + `.AddServiceToken("businessrules")`
  - Consumers subscribe with `eventBus.AddSubscription<BusinessRulesSnapshotPublishedIntegrationEvent, BusinessRulesSnapshotPublishedIntegrationEventHandler>()`

- [ ] **Step 1: Write failing tests** (fake `IRulesSnapshotSource` via NSubstitute; `HttpMessageHandler` stub for HTTP tests)
  - `RefreshReplacesSnapshotWhenSourceReturnsNewerVersion`: `Current.Version == 3`, `LastRefreshedAt == now`
  - `RefreshKeepsLastSnapshotWhenSourceThrows`: v3 stays, returns `false`, no exception
  - `RefreshIgnoresOlderVersion`: v3 cached, source returns v2 → stays v3
  - `CacheStartsWithEmptySnapshot`: `Current.Version == 0`
  - `HttpSourceSendsIfNoneMatchAndMapsNotModified`: request header `"3"`, 304 → `NotModified == true`
  - `HandlerSkipsRefreshWhenEventVersionNotNewer`: source not called when event v3 and cache v3
  - `ServiceTokenHandlerReusesTokenUntilNearExpiry`: two requests → one token call; after time advances past `expires_in − 60` → second token call (inject `TimeProvider` substitute)
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement** the members listed above.
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): add snapshot cache, refresh and service token handler`

### Task 4: Identity.API — scopes, clients, roles

**Files:**
- Modify: `src/Identity.API/Configuration/Config.cs` (`GetApis`, `GetApiScopes`, `GetClients`)
- Modify: `src/Identity.API/Services/ProfileService.cs` (role claims)
- Modify: `src/Identity.API/UsersSeed.cs` (roles + assignments, D8)
- Modify: `.config/CredScanSuppressions.json` only if a new secret placeholder appears in config
- Test: `tests/Application.UnitTests/IdentityConfigurationTests.cs` (existing file, MSTest)

**Interfaces:**
- Produces: API resources `businessrules` (scopes `businessrules`, `businessrules.admin`) and `ordering.usage` → scope `ordering.usage`. `ApiScope("businessrules.admin") { UserClaims = { "role" } }`. Clients:
  - `businessrules-reader`: `ClientCredentials`, secret = config `BusinessRulesReaderSecret`, scopes `businessrules`
  - `businessrules-api`: `ClientCredentials`, secret = config `BusinessRulesApiSecret`, scopes `ordering.usage`
  - `businessrulesswaggerui`: same shape as `orderingswaggerui` (`Config.cs:161`), redirect from `BusinessRulesApiClient`, scopes `businessrules.admin`
  - `eshopadmin`: `Code` + `RequirePkce = true`, `RequireClientSecret = false`, redirect `{EShopAdminClient}/signin-oidc`, scopes `openid profile businessrules.admin`. Registered only when `EShopAdminClient` is configured.
- Roles `RulesEditor`, `RulesPublisher`; token claim type `role`.

- [ ] **Step 1: Write failing tests**
  - `BusinessRulesScopesAreDefined`: `GetApiScopes()` contains `businessrules`, `businessrules.admin`, and `ordering.usage`, and the admin scope's `UserClaims` contains `"role"`
  - `ServiceClientsUseClientCredentialsWithLeastPrivilege`: `businessrules-reader.AllowedScopes` = exactly `["businessrules"]`; `businessrules-api.AllowedScopes` = exactly `["ordering.usage"]`; both `GrantTypes.ClientCredentials`
  - `EShopAdminClientRequiresPkceAndAdminScopeOnly`: no `orders`/`basket` scope (customer scopes never reused)
  - `EShopAdminClientOmittedWhenNotConfigured`
  - `ProfileServiceAddsRoleClaimsWhenRequested`: substitute `UserManager<ApplicationUser>` returns roles `["RulesPublisher"]`; when `RequestedClaimTypes` contains `role`, the issued claims contain `role=RulesPublisher`
- [ ] **Step 2: Run** `dotnet test --project tests/Application.UnitTests --filter "FullyQualifiedName~IdentityConfigurationTests"`. Expected: FAIL.
- [ ] **Step 3: Implement.** Seed: create both roles if missing and assign `alice → RulesPublisher`, `bob → RulesEditor` (idempotent). `ProfileService`: add `role` claims from `GetRolesAsync` only when they are requested.
- [ ] **Step 4: Run** the same command. Expected: PASS.
- [ ] **Step 5: Commit** `feat(identity): add business rules scopes, service clients and admin roles`

### Task 5: BusinessRules.API skeleton + AppHost wiring

**Files:**
- Create: `src/BusinessRules.API/BusinessRules.API.csproj` (copy `Catalog.API.csproj` refs: `EventBusRabbitMQ`, `IntegrationEventLogEF`, `eShop.ServiceDefaults`; plus `eShop.BusinessRules.Client`; packages `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL`, `Asp.Versioning.Http`, `FluentValidation.DependencyInjectionExtensions`)
- Create: `src/BusinessRules.API/{Program.cs,GlobalUsings.cs,appsettings.json,Properties/launchSettings.json}`. `appsettings.json` has `Identity:Audience = "businessrules"`, `EventBus:SubscriptionClientName = "BusinessRules"`, `ServiceClient:ClientId = "businessrules-api"`, `OpenApi` block as in Ordering (ClientId `businessrulesswaggerui`).
- Create: `src/BusinessRules.API/Extensions/Extensions.cs` (`AddApplicationServices`), `Infrastructure/{RulesContext,RulesContextSeed}.cs`, `Authorization/RulesPolicies.cs`
- Modify: `src/eShop.AppHost/AppHost.cs`
- Test: `tests/eShop.AppHost.UnitTests/BusinessRulesResourceTests.cs`

**Interfaces:**
- Produces: `RulesPolicies.Runtime` (claim `scope`=`businessrules`), `RulesPolicies.Editor` (scope `businessrules.admin` AND role `RulesEditor` or `RulesPublisher`), `RulesPolicies.Publisher` (scope AND role `RulesPublisher`). `RulesContext : DbContext` with default schema `rules` and `UseIntegrationEventLogs()`. Aspire resource names `rulesdb` and `businessrules-api`.
- AppHost (after `orderingApi` is declared):

```csharp
var rulesDb = postgres.AddDatabase("rulesdb");
var rulesReaderSecret = builder.AddParameter("businessrules-reader-secret", new GenerateParameterDefault { MinLength = 32, Special = false }, secret: true, persist: true);
var rulesApiSecret    = builder.AddParameter("businessrules-api-secret",    new GenerateParameterDefault { MinLength = 32, Special = false }, secret: true, persist: true);
var businessRulesApi = builder.AddDotnetProject("businessrules-api", "../BusinessRules.API")
    .WithReference(rabbitMq).WaitFor(rabbitMq)
    .WithReference(rulesDb).WaitFor(rulesDb)
    .WithReference(orderingApi)                     // usage guard (D3); no WaitFor
    .WithHttpHealthCheck("/health")
    .WithEnvironment("Identity__Url", identityEndpoint)
    .WithEnvironment("ServiceClient__ClientSecret", rulesApiSecret);
orderingApi.WithReference(businessRulesApi)          // NO WaitFor: T1 must not depend on T3
    .WithEnvironment("ServiceClient__ClientSecret", rulesReaderSecret);
identityApi.WithEnvironment("BusinessRulesApiClient", businessRulesApi.GetEndpoint("http"))
    .WithEnvironment("BusinessRulesReaderSecret", rulesReaderSecret)
    .WithEnvironment("BusinessRulesApiSecret", rulesApiSecret);
```

- [ ] **Step 1: Write failing AppHost tests** (pattern: `AppHostConfigurationTests.cs`)
  - `BusinessRulesApiReferencesRulesDbAndEventBus`
  - `OrderingApiDoesNotWaitForBusinessRulesApi`: no `WaitAnnotation` on `ordering-api` targeting `businessrules-api`
- [ ] **Step 2: Run** `dotnet test --project tests/eShop.AppHost.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement** the skeleton (Program mirrors `Catalog.API/Program.cs:3-29`; `AddMigration<RulesContext, RulesContextSeed>()`; health checks come from Aspire Npgsql + RabbitMQ) and the AppHost lines above. Add the projects to `eShop.slnx` and `eShop.Web.slnf`.
- [ ] **Step 4: Run** the same command, plus `dotnet build eShop.Web.slnf`. Expected: PASS, 0 warnings.
- [ ] **Step 5: Commit** `feat(businessrules): scaffold BusinessRules.API and wire rulesdb in AppHost`

### Task 6: Rule versions, drafts, history, audit (admin API)

**Files:**
- Create: `src/BusinessRules.API/Model/{RuleVersion,RuleVersionStatus,AuditEntry,AuditAction}.cs`, `Infrastructure/EntityConfigurations/*.cs`, `Infrastructure/Migrations/*_InitialRules.cs`
- Create: `src/BusinessRules.API/Apis/RulesAdminApi.cs`, `Apis/RulesDtos.cs`, `Validation/{CreateDraftRequestValidator,UpdateDraftRequestValidator}.cs`
- Create: `tests/BusinessRules.FunctionalTests/{BusinessRules.FunctionalTests.csproj,BusinessRulesApiFixture.cs,TestAuthHandler.cs,RulesAdminApiTests.cs}` (MSTest.Sdk; fixture modelled on `tests/Ordering.FunctionalTests/OrderingApiFixture.cs` with MSTest `[AssemblyInitialize]`. `TestAuthHandler` reads `x-test-scopes`, `x-test-roles`, `x-test-sub`. `TestRuleType` registered via `AddRuleType<TestRuleType>()`.)
- Test: `tests/BusinessRules.UnitTests/RuleVersionTests.cs`

**Interfaces:**
- Produces:
  - `RuleVersion` (EF entity with behaviour): `Guid Id, string RuleTypeId, string Scope, int VersionNumber, JsonDocument Parameters (jsonb), DateTimeOffset EffectiveFrom, DateTimeOffset? EffectiveTo, RuleVersionStatus Status { Draft, Published, Superseded }, string CreatedBy, DateTimeOffset CreatedAt, string? PublishedBy, DateTimeOffset? PublishedAt`; methods `UpdateDraft(JsonDocument, DateTimeOffset from, DateTimeOffset? to)`, `Publish(string actor, DateTimeOffset now)` (clamps `EffectiveFrom` per D5), `Supersede()`. Editing or publishing a non-draft throws `InvalidOperationException`. Unique index `(RuleTypeId, Scope, VersionNumber)`.
  - `AuditEntry(long Id, DateTimeOffset OccurredAt, string ActorId, AuditAction Action { DraftCreated, DraftUpdated, Published, GuardOverridden }, string RuleTypeId, string Scope, Guid? RuleVersionId, string? BeforeJson, string? AfterJson, string? Reason)`
  - Routes (`NewVersionedApi("Rules")`, group `api/rules`, v1):
    - `GET types`: `Ok<IReadOnlyList<RuleTypeDto>>` (Editor)
    - `GET versions?ruleTypeId=&scope=`: history, newest first (Editor)
    - `POST versions`: `Results<Created<RuleVersionDto>, ValidationProblem>` (Editor)
    - `PUT versions/{id:guid}`: `Results<Ok<RuleVersionDto>, NotFound, ValidationProblem, Conflict<ProblemDetails>>` (Editor; 409 if not draft)
    - `GET audit?pageIndex=&pageSize=`: `Ok<PaginatedItems<AuditEntryDto>>` (Editor)
  - Validation: rule type exists; `IRuleType.Validate` errors → 400; `EffectiveTo > EffectiveFrom`; scope = `default` only (spec non-goal: per-offering scoping).
- **Retention (data-retention.md):** `RuleVersion` and `AuditEntry` kept **7 years** after superseded (spec *Data*, marked as a call). Add both to `standards/data-retention.md` in Task 15.

- [ ] **Step 1: Write failing tests**
  - Unit: `PublishedVersionCannotBeEdited`, `PublishClampsPastEffectiveFromToNow`, `PublishSetsPublisherAndTimestamp`
  - Functional: `CreateDraftWithValidParametersReturnsCreatedAndAudits` (201, `VersionNumber == 1`, audit row `DraftCreated`)
  - `CreateDraftWithInvalidParametersReturnsValidationProblem` (400, error key from `TestRuleType.Validate`)
  - `CreateDraftForUnknownRuleTypeReturnsValidationProblem`
  - `UpdatePublishedVersionReturnsConflict`
  - `AdminEndpointsRequireAdminScope` (token with only `businessrules` → 403)
  - `CustomerTokenIsRejected` (scope `orders` → 403)
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests` and `dotnet test --project tests/BusinessRules.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement.** Create the migration with `dotnet ef migrations add InitialRules --project src/BusinessRules.API --context RulesContext`. Actor = `sub` claim. Log `"Rule draft {RuleTypeId} v{VersionNumber} created"` with no actor (no user ids in logs).
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): add rule version drafts, history and audit`

### Task 7: Snapshot endpoint (runtime)

**Files:**
- Create: `src/BusinessRules.API/Model/RulesSnapshotRecord.cs` (`long Version` PK, `DateTimeOffset PublishedAt`, `JsonDocument Content`), `Apis/RulesRuntimeApi.cs`
- Test: `tests/BusinessRules.FunctionalTests/RulesSnapshotApiTests.cs`

**Interfaces:**
- Produces: `GET api/rules/snapshot` (Runtime policy) → `Results<Ok<RulesSnapshot>, StatusCodeHttpResult>` with `ETag: "<version>"`. `If-None-Match` equal to the latest → 304. With no snapshot rows → 200 `RulesSnapshot.Empty`. Content is serialized with the same `JsonSerializerOptions` as `HttpRulesSnapshotSource` (expose `RulesSnapshotJson.Options` from the client lib).
- **Retention:** `RulesSnapshotRecord` latest + all referenced by orders for 7 years (spec *Data*, a call).

- [ ] **Step 1: Write failing tests**
  - `SnapshotReturnsEmptyVersionZeroBeforeFirstPublish`
  - `SnapshotReturnsNotModifiedWhenEtagMatches`
  - `SnapshotRequiresRuntimeScope` (admin-only token → 403; anonymous → 401)
  - `SnapshotRoundTripsThroughHttpRulesSnapshotSource`: the client-lib source, pointed at the factory's `HttpClient`, deserializes the version and rules
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement** `RulesRuntimeApi.MapRulesRuntimeApiV1()`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): serve rules snapshot with ETag`

### Task 8: Publish → snapshot → outbox event

**Files:**
- Create: `src/BusinessRules.API/Services/{RulePublishingService,RulesSnapshotBuilder,IRulesIntegrationEventService,RulesIntegrationEventService,BusinessRulesMetrics}.cs` (outbox pattern copied from Catalog.API's integration event service)
- Modify: `src/BusinessRules.API/Apis/RulesAdminApi.cs` (`POST versions/{id:guid}/publish`)
- Test: `tests/BusinessRules.UnitTests/RulesSnapshotBuilderTests.cs`, `tests/BusinessRules.FunctionalTests/PublishApiTests.cs`

**Interfaces:**
- Consumes: `RuleVersion.Publish/Supersede` (Task 6), `BusinessRulesSnapshotPublishedIntegrationEvent` (Task 3), `IPublishGuard` (Task 10; register `AllowAllPublishGuard` here and replace it in Task 10).
- Produces:
  - `RulesSnapshotBuilder.Build(long version, DateTimeOffset publishedAt, IEnumerable<RuleVersion> published, DateTimeOffset now) : (RulesSnapshot Snapshot, IReadOnlyList<Guid> SupersededIds)` implements D5
  - `POST api/rules/versions/{id}/publish` body `PublishRequest(string? GuardOverrideReason)` → `Results<Ok<PublishResultDto>, NotFound, Conflict<ProblemDetails>, ForbidHttpResult, ProblemHttpResult>` (Publisher policy). `PublishResultDto(Guid RuleVersionId, long SnapshotVersion)`.
  - In one transaction: publish the version, supersede the losers, insert `RulesSnapshotRecord(version = max + 1)`, audit `Published`, and save the outbox event `BusinessRulesSnapshotPublishedIntegrationEvent(version, publishedAt)`. Publish to the bus after commit. A unique-key violation on the snapshot version → 409 `"concurrent-publish"`.
  - Counter `rules.publish.count` on meter `eShop.BusinessRules`

- [ ] **Step 1: Write failing tests**
  - Unit: `BuildIncludesFutureDatedVersions`, `BuildSupersedesOlderOpenEndedVersionOnceNewerIsEffective`, `BuildKeepsOlderVersionWhenNewerHasEffectiveTo`
  - Functional: `PublishCreatesNextSnapshotVersionAndOutboxEvent` (snapshot v1 → v2; `IntegrationEventLog` row with `EventTypeName` ending `BusinessRulesSnapshotPublishedIntegrationEvent`; audit `Published`)
  - `EditorCannotPublish` (role `RulesEditor` → 403, spec AC)
  - `PublisherCanPublishOwnDraft` (no second-person approval, spec decision)
  - `PublishingNonDraftReturnsConflict`
  - `ConcurrentPublishProducesDistinctSnapshotVersions` (Review Focus 4): publish two drafts in parallel; snapshot versions are distinct and consecutive, or exactly one call returns 409
- [ ] **Step 2: Run** both test projects. Expected: FAIL.
- [ ] **Step 3: Implement.** Use `ResilientTransaction`-style execution as in Catalog's integration event service. Log `"Rules snapshot {SnapshotVersion} published"`.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): publish rule versions into versioned snapshots via outbox`

### Task 9: Ordering — v1 usage counter + usage endpoint + metrics

**Files:**
- Create: `src/Ordering.Infrastructure/Usage/{ApiUsageDaily,IApiUsageRecorder,ApiUsageRecorder}.cs`, `EntityConfigurations/ApiUsageDailyEntityTypeConfiguration.cs`
- Create: `src/Ordering.API/Apis/UsageApi.cs`, `src/Ordering.API/Infrastructure/OrderingMetrics.cs`
- Modify: `src/Ordering.Infrastructure/OrderingContext.cs` (DbSet), `src/Ordering.API/Program.cs`, `Extensions/Extensions.cs` (policy `OrderingPolicies.UsageReader` = claim `scope`=`ordering.usage`)
- Test: `tests/Ordering.FunctionalTests/UsageApiTests.cs`

**Interfaces:**
- Produces:
  - Table `ordering.apiusagedaily(Day date, Operation varchar(64), Calls bigint, PK(Day, Operation))`. **Retention 90 days**: `RecordAsync` also deletes rows older than 90 days. Non-personal.
  - `IApiUsageRecorder { Task RecordAsync(string operation, CancellationToken ct); Task<long> GetCallsAsync(string operation, int days, CancellationToken ct); }`. Upsert via `INSERT … ON CONFLICT ("Day","Operation") DO UPDATE SET "Calls" = apiusagedaily."Calls" + 1`.
  - `const string CreateOrderV1Operation = "ordering.create_order.v1.calls"`
  - `GET api/usage/create-order-v1?days=7` (v1, `NewVersionedApi("Usage")`, policy UsageReader; `days` 1–30) → `Ok<ApiUsageResponse(string Operation, int Days, long Calls)>`
  - `OrderingMetrics` (meter `eShop.Ordering`): counters `ordering.create_order.v1.calls` and `pricing.total_mismatch.count`; histogram `pricing.quote.duration` (ms)
- Migration: added in Task 11's migration (one Ordering migration for phase 1).

- [ ] **Step 1: Write failing tests:** `UsageReturnsZeroWhenNoV1Calls`, `UsageCountsCallsWithinWindowOnly` (seed rows at −8 d and −1 d, `days=7` → 1 day counted), `UsageRequiresUsageScope` (customer `orders` scope → 403). Fake auth: extend `AutoAuthorizeMiddleware` to add `scope` claims from header `x-test-scopes` when present.
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(ordering): record create-order v1 usage and expose usage endpoint`

### Task 10: BusinessRules.API — price-rule publish guard

**Files:**
- Create: `src/BusinessRules.API/Services/{IPublishGuard,V1ClientsPublishGuard,IOrderingUsageClient,OrderingUsageClient}.cs`
- Modify: `Extensions/Extensions.cs` (typed client `https+http://ordering-api`, `.AddApiVersion(1.0)` pattern from WebApp, `.AddServiceToken("ordering.usage")`)
- Test: `tests/BusinessRules.UnitTests/V1ClientsPublishGuardTests.cs`, `tests/BusinessRules.FunctionalTests/PublishGuardApiTests.cs`

**Interfaces:**
- Consumes: `GET api/usage/create-order-v1` (Task 9), `IRuleType.AffectsPrice` (Task 2), publish endpoint (Task 8).
- Produces: `IPublishGuard { Task<GuardResult> CheckAsync(IRuleType type, string? overrideReason, CancellationToken ct); }`; `GuardResult(bool Allowed, bool Overridden, string? ProblemType)`. Rules:
  - `!AffectsPrice` → allowed
  - calls = 0 → allowed
  - calls > 0 and no reason → 409 ProblemDetails `type = "https://eshop/problems/v1-clients-active"`, detail includes the call count
  - calls > 0 and reason (1–500 chars) → allowed + audit `GuardOverridden` with the reason
  - usage call fails → 503 ProblemDetails (fail closed, D3)

- [ ] **Step 1: Write failing tests** (unit with substitute `IOrderingUsageClient`): `NonPriceRulePublishesWithoutUsageCheck` (client not called), `PriceRuleBlockedWhileV1CallsInLast7Days`, `PriceRulePublishesWithOverrideReasonAndAudits`, `PriceRuleBlockedWhenUsageUnavailable`, `OverrideReasonLongerThan500CharsIsRejected`. Functional: `PublishPriceRuleWithV1TrafficReturnsConflict` (fixture replaces `IOrderingUsageClient`; `TestRuleType` variant with `AffectsPrice = true`).
- [ ] **Step 2: Run** both projects. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): guard price-affecting publishes while v1 clients are active`

### Task 11: Ordering domain — pricing fields, `GetTotal` fix, migration + backfill, read models

**Files:**
- Modify: `src/Ordering.Domain/AggregatesModel/OrderAggregate/Order.cs` (fields; `AddOrderItem` at line 88; `GetTotal` at line 185)
- Modify: `src/Ordering.Domain/AggregatesModel/OrderAggregate/OrderItem.cs` (`BasePrice`)
- Modify: `src/Ordering.Infrastructure/EntityConfigurations/{OrderEntityTypeConfiguration,OrderItemEntityTypeConfiguration}.cs`
- Create: `src/Ordering.Infrastructure/Migrations/<ts>_AddOrderPricing.cs` (includes `apiusagedaily` from Task 9)
- Modify: `src/Ordering.API/Application/Queries/{OrderViewModel,OrderQueries}.cs`
- Test: `tests/Ordering.UnitTests/Domain/OrderAggregateTest.cs`, `Domain/Builders.cs`; `tests/Ordering.FunctionalTests/OrderingApiTests.cs`

**Interfaces:**
- Produces:
  - `Order`: `decimal Subtotal`, `decimal DiscountTotal`, `decimal ShippingFee`, `decimal Total`, `long PricingSnapshotVersion` (private setters)
  - `Order.AddOrderItem(int productId, string productName, decimal unitPrice, decimal discount, string pictureUrl, int units = 1, decimal? basePrice = null)`. `basePrice` defaults to `unitPrice`.
  - `Order.ApplyPricing(decimal shippingFee, long pricingSnapshotVersion)`: recomputes `Subtotal = Σ units×unitPrice`, `DiscountTotal = Σ discount`, `Total = Subtotal − DiscountTotal + shippingFee`. Throws `OrderingDomainException` if `shippingFee < 0` or `Total < 0`.
  - `Order.GetTotal()` = `_orderItems.Sum(o => o.Units * o.UnitPrice - o.Discount) + ShippingFee`
  - `OrderItem.BasePrice` (decimal); ctor validates `unitPrice ≥ 0.01` in addition to ORD-010
  - Read model `Order`: add `Subtotal`, `DiscountTotal`, `ShippingFee`; `Total` from the column. `Orderitem`: add `Discount`. `OrderSummary.Total` reads `o.Total` (replaces the SQL `Sum`).
- Migration backfill (`migrationBuilder.Sql`): `orderItems."BasePrice" = "UnitPrice"`; `orders."Subtotal" = Σ units×unit_price`, `"DiscountTotal" = Σ discount`, `"ShippingFee" = 0`, `"Total" = Subtotal − DiscountTotal`, `"PricingSnapshotVersion" = 0`.
- **Retention:** new order fields follow the ADR-0006 order tiers; they are financial and kept in the archived tier.

- [ ] **Step 1: Write failing tests**
  - `GetTotalSubtractsDiscountAndAddsShipping`: 2×10 discount 3, 1×5, shipping 4.99 → 26.99
  - `ApplyPricingSetsBreakdown` (Subtotal 25, DiscountTotal 3, Total 26.99, snapshot 7)
  - `ApplyPricingRejectsNegativeShipping`
  - `AddOrderItemDefaultsBasePriceToUnitPrice`
  - `OrderItemRejectsUnitPriceBelowOneCent`
  - Functional: `GetOrderReturnsPricingBreakdown`
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.UnitTests` and `dotnet test --project tests/Ordering.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement.** Create the migration: `dotnet ef migrations add AddOrderPricing --startup-project src/Ordering.API --project src/Ordering.Infrastructure --context OrderingContext` (command from `OrderingContext.cs:6-8`). Add the backfill SQL to `Up`.
- [ ] **Step 4: Run** both. Expected: PASS. Existing aggregate tests stay green.
- [ ] **Step 5: Commit** `feat(ordering): store order pricing breakdown and fix GetTotal`

### Task 12: Ordering — catalog prices, pricing service, quote endpoint

**Files:**
- Create: `src/Ordering.API/Application/Pricing/{ICatalogPriceSource,CatalogPriceClient,IOrderPricingService,OrderPricingService}.cs`
- Create: `src/Ordering.API/Apis/PricingApi.cs`
- Modify: `src/Ordering.API/Program.cs` (`app.NewVersionedApi("Pricing").MapPricingApiV1().AllowAnonymous()`), `Extensions/Extensions.cs` (`builder.AddBusinessRulesClient()`, subscription, typed client), `src/Ordering.API/appsettings.json` (`ServiceClient:ClientId = "businessrules-reader"`)
- Modify: `src/eShop.AppHost/AppHost.cs`: `orderingApi.WithReference(catalogApi)`
- Modify: `tests/Ordering.FunctionalTests/OrderingApiFixture.cs`: register `FakeCatalogPriceSource` (products 1–5 at fixed prices) and a substitute `IRulesSnapshotSource`
- Test: `tests/Ordering.FunctionalTests/PricingApiTests.cs`, `tests/Ordering.UnitTests/Application/OrderPricingServiceTests.cs`

**Interfaces:**
- Consumes: `PricingCalculator`, `RulesSnapshotCache` (Tasks 2–3); Catalog `GET api/catalog/items/by?ids=` (`CatalogApi.cs:32`, returns `List<CatalogItem>` with `Price`, `CatalogTypeId`, `CatalogBrandId`).
- Produces:
  - `ICatalogPriceSource { Task<IReadOnlyDictionary<int, CatalogPrice>> GetPricesAsync(IReadOnlyCollection<int> productIds, CancellationToken ct); }`. `CatalogPriceClient` uses base `https+http://catalog-api` and `api-version=1.0` (resilience from ServiceDefaults `Extensions.cs:25`).
  - `IOrderPricingService { Task<PriceQuote> QuoteAsync(IReadOnlyList<QuoteLine> lines, CancellationToken ct); }`. Uses `cache.Current`, `TimeProvider.GetUtcNow()`, and records `pricing.quote.duration`.
  - `POST api/pricing/quote` (v1, anonymous or `orders`) body `PriceQuoteRequest(List<QuoteLine> Items)` (1–100 lines) → `Results<Ok<PriceQuote>, ValidationProblem, ProblemHttpResult>`. Unknown products → 400 with `errors["items"]` listing the ids. Catalog unavailable → 503 ProblemDetails.

- [ ] **Step 1: Write failing tests**
  - `QuoteReturnsCatalogPricesWithDefaults` (2×product 1 @ 10.00 → Total 20.00, `SnapshotVersion` 0)
  - `QuoteAllowsAnonymousCaller`
  - `QuoteWithUnknownProductReturnsBadRequest`
  - `QuoteWithMoreThan100LinesReturnsBadRequest`
  - `QuoteSucceedsWithDefaultsWhenSnapshotSourceUnavailable` (Review Focus 1: source throws on every call)
  - Unit: `QuoteUsesCurrentSnapshotVersion` (cache at v5 → quote v5)
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests` and `tests/Ordering.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(ordering): price baskets server-side and add quote endpoint`

### Task 13: Ordering — create order v2 (409) and v1 server pricing

**Files:**
- Modify: `src/Ordering.API/Apis/OrdersApi.cs` (add `MapOrdersApiV2`, `CreateOrderV2Async`, `CreateOrderV2Request`; change `CreateOrderAsync` at line 118)
- Modify: `src/Ordering.API/Program.cs` (`orders.MapOrdersApiV2().RequireAuthorization()`)
- Modify: `src/Ordering.API/Application/Commands/{CreateOrderCommand,CreateOrderCommandHandler}.cs`, `Application/Models/OrderItemDTO` (add `BasePrice`)
- Modify: `src/Ordering.API/Extensions/BasketItemExtensions.cs` (only `CreateOrderDraftCommandHandler` uses it after this task)
- Test: `tests/Ordering.FunctionalTests/OrderingApiTests.cs`, `tests/Ordering.UnitTests/Application/NewOrderRequestHandlerTest.cs` (update ctor usage)

**Interfaces:**
- Consumes: `IOrderPricingService` (Task 12), `Order.ApplyPricing` (Task 11), `IApiUsageRecorder` + `OrderingMetrics` (Task 9).
- Produces:
  - `CreateOrderCommand(PriceQuote quote, string userId, string userName, string city, string street, string state, string country, string zipcode, string cardNumber, string cardHolderName, DateTime cardExpiration, string cardSecurityNumber, int cardTypeId)`. `OrderItems` come from `quote.Lines`. New properties `ShippingFee`, `PricingSnapshotVersion`. The handler passes `basePrice` and `discount` from each line, then calls `order.ApplyPricing(...)`.
  - `record CreateOrderV2Request(string UserId, string UserName, string City, string Street, string State, string Country, string ZipCode, string CardNumber, string CardHolderName, DateTime CardExpiration, string CardSecurityNumber, int CardTypeId, string Buyer, List<QuoteLine> Items, decimal ExpectedTotal)`
  - v2 `POST api/orders` → `Results<Ok, BadRequest<string>, ValidationProblem, ProblemHttpResult>`. If `quote.Total != PriceRounding.Round(ExpectedTotal)` → **409** ProblemDetails `type = "https://eshop/problems/prices-changed"`, `extensions["quote"] = quote`, increment `pricing.total_mismatch.count`, and send **no** command (so no `ClientRequest` is written; Review Focus 5).
  - v1 `POST api/orders`: builds `QuoteLine`s from `Items` (`ProductId`, `Quantity`), **ignores** `UnitPrice`/`OldUnitPrice`, never returns 409, and calls `usage.RecordAsync(CreateOrderV1Operation)` + counter.
  - `MapOrdersApiV2` maps the same routes as v1 (D6), except `POST /`.

- [ ] **Step 1: Write failing tests**
  - `CreateOrderV2WithMatchingExpectedTotalCreatesOrderAtServerPrices` (order `Total` == quote total)
  - `CreateOrderV2WithStaleExpectedTotalReturnsConflictWithFreshQuote` (409; body `quote.total` = server value; no order)
  - `CreateOrderV2WithUnknownProductReturnsBadRequest` (Review Focus 2)
  - `RetryAfterPriceConflictWithSameRequestIdCreatesOrder` (Review Focus 5)
  - `CreateOrderV1IgnoresClientUnitPrice` (client sends `UnitPrice` 0.01 → order charged the catalog price; spec AC "tampered price")
  - `CreateOrderV1RecordsUsage` (usage endpoint returns 1)
  - Update existing `AddNewOrder` to use fake-catalog product ids.
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests` and `tests/Ordering.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.** Keep the masked-card logging behaviour from `OrdersApi.cs:139`. Log scope gets `order.id` and `rules.snapshot.version` only.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(ordering): add create order v2 with expected total and server-price v1`

### Task 14: mobile-bff routes

**Files:**
- Modify: `src/eShop.AppHost/Extensions.cs` `ConfigureMobileBffRoutes` (≈ lines 91–140): orders route values `["1.0", "1", "2.0"]`; new route `/api/pricing/quote` → ordering cluster, `api-version` `["1.0", "1"]`. Do **not** add any `businessrules-api` route (snapshot is internal-only, spec *Security*).
- Test: `tests/eShop.AppHost.UnitTests/MobileBffRoutesTests.cs`

- [ ] **Step 1: Write failing tests:** `MobileBffRoutesPricingQuoteToOrdering`, `MobileBffOrdersRouteAcceptsVersion2`, `MobileBffHasNoBusinessRulesRoute`.
- [ ] **Step 2: Run** `dotnet test --project tests/eShop.AppHost.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(apphost): route pricing quote and orders v2 through mobile-bff`

### Task 15: WebApp — quote-based prices and v2 checkout

**Files:**
- Create: `src/WebAppComponents/Services/IProductPriceProvider.cs`
- Modify: `src/WebAppComponents/Catalog/CatalogListItem.razor` (lines 7, 18: `DisplayPrice ?? Item.Price`)
- Create: `src/WebApp/Services/{PricingService,QuotePriceProvider,PricesChangedException}.cs`
- Modify: `src/WebApp/Services/{BasketState,OrderingService}.cs`, `src/WebApp/Extensions/Extensions.cs` (≈ lines 31–40), `src/WebApp/WebApp.csproj` (ref `eShop.BusinessRules.Client`)
- Modify: `src/WebApp/Components/Pages/{Catalog/Catalog.razor:88, Item/ItemPage.razor:41, Cart/CartPage.razor:46,79,95,123, Checkout/Checkout.razor:114}`
- Modify: `tests/Application.UnitTests/Application.UnitTests.csproj` (ref WebApp)
- Test: `tests/Application.UnitTests/WebAppPricingTests.cs`; E2E `e2e/CheckoutTest.spec.ts`

**Interfaces:**
- Consumes: `POST api/pricing/quote` v1, `POST api/orders` v2 (Tasks 12–13), `PriceQuote`/`QuoteLine` types (Task 2).
- Produces:
  - `interface IProductPriceProvider { Task<IReadOnlyDictionary<int, decimal>> GetDisplayPricesAsync(IReadOnlyCollection<int> productIds); }`. Display price = `PriceRounding.Round(line.LineTotal / line.Units)` for a 1-unit quote line. `CatalogListItem` gets `[Parameter] public decimal? DisplayPrice { get; set; }`.
  - `PricingService(HttpClient)` (`https+http://ordering-api`, `.AddApiVersion(1.0)`, `.AddAuthToken()`): `Task<PriceQuote> GetQuoteAsync(IReadOnlyList<QuoteLine> lines)`
  - `QuotePriceProvider`: on failure, returns an empty dictionary and logs a warning (D7)
  - `OrderingService`: typed client switches to `.AddApiVersion(2.0)`. `CreateOrder(CreateOrderV2Request, Guid requestId)` throws `PricesChangedException(PriceQuote Quote)` on 409. Config `Ordering:UseCreateOrderV1` (default `false`) sends the v1 body instead (rollback path, spec *Rollout*).
  - `BasketState`: `Task<PriceQuote?> GetQuoteAsync()` (cached with the basket and invalidated on change); `BasketItem.UnitPrice` filled from the quote line; `CheckoutAsync(BasketCheckoutInfo)` sends `ExpectedTotal = checkoutInfo.ExpectedTotal` (new property, set by Checkout from the displayed quote) and deletes the basket only on success.
  - Checkout shows the subtotal, discount, shipping, and total from the quote. On `PricesChangedException`, it shows the exact copy *"Prices have changed since you reviewed your order. Check the new total and place your order again."*, replaces the displayed quote, and keeps the basket.
  - Cart totals come from the quote (remove the client-side `Sum` at `CartPage.razor:123`).

- [ ] **Step 1: Write failing tests**
  - Unit (stub `HttpMessageHandler`): `OrderingServiceThrowsPricesChangedWithFreshQuoteOn409`, `OrderingServiceSendsApiVersion2AndRequestId`, `QuotePriceProviderReturnsEmptyOnFailure`, `QuotePriceProviderUsesPerUnitPriceAfterDiscount`
  - E2E: extend `e2e/CheckoutTest.spec.ts` to assert that the order total on `/user/orders` equals the total shown on checkout
- [ ] **Step 2: Run** `dotnet test --project tests/Application.UnitTests --filter "FullyQualifiedName~WebAppPricingTests"`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the unit tests, then `npm run test:e2e`. Expected: PASS, and existing `BrowseItemTest`, `CartTest`, and `CheckoutTest` stay green (J1–J3).
- [ ] **Step 5: Commit** `feat(webapp): show server-quoted prices and checkout with create order v2`

### Task 16: ClientApp (MAUI) — quote + v2

**Files:**
- Create: `src/ClientApp/Models/Pricing/{PriceQuote,PricedLine,QuoteLine}.cs`, `src/ClientApp/Services/Order/CreateOrderResult.cs`
- Modify: `src/ClientApp/Services/Order/{IOrderService,OrderService}.cs` (`ApiVersion` → `"api-version=2.0"`), the mock order service, `src/ClientApp/ViewModels/CheckoutViewModel.cs` (lines 106–154)
- Test: `tests/ClientApp.UnitTests/Services/OrdersServiceTests.cs`, `ViewModels/CheckoutViewModelTests.cs` (new)

**Interfaces:**
- Produces:
  - `IOrderService.GetQuoteAsync(IEnumerable<QuoteLine> lines) : Task<PriceQuote>` (`{GatewayOrdersEndpointBase}/api/pricing/quote?api-version=1.0`)
  - `IOrderService.CreateOrderAsync(Order newOrder, decimal expectedTotal) : Task<CreateOrderResult>`, where `abstract record CreateOrderResult { sealed record Created; sealed record PricesChanged(PriceQuote Quote); }`
  - `CheckoutViewModel` loads the quote on navigation and binds `Total` to `quote.Total`. On `PricesChanged`, it shows the dialog with the same copy as Task 15, updates `Total`, and does **not** clear the basket.

- [ ] **Step 1: Write failing tests:** `CheckoutUsesQuotedTotal`, `CheckoutOnPricesChangedShowsDialogAndKeepsBasket` (`MockDialogService` records the message), `OrderServiceUsesApiVersion2ForCreate`.
- [ ] **Step 2: Run** `dotnet test --project tests/ClientApp.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(clientapp): use server quotes and create order v2`

### Task 17: Resilience check + architecture docs (eShopOrgMeta)

**Files:**
- Test: `tests/Ordering.FunctionalTests/ResilienceTests.cs`
- Modify (eShopOrgMeta): `architecture/service-catalog.md` (BusinessRules.API row: T3, `rulesdb`, publishes the snapshot event; Ordering.API now calls Catalog.API), `architecture/integration-events.md` (move `BusinessRulesSnapshotPublishedIntegrationEvent` to the main table with consumer Ordering.API; `OrderPricedIntegrationEvent` stays Proposed), `architecture/system-context.md` (diagram), `architecture/environments-and-deployment.md` (`rulesdb`, service-client secrets per environment, managed in Key Vault when deployed), `architecture/domain-glossary.md` (move *rule type, rule version, rules snapshot, price quote* to adopted), `architecture/business-rules-catalog.md` (G-03 note: store/audit/effective-dating exist; rule types arrive in phase 2), `standards/data-retention.md` (rows: `RuleVersion`, `AuditEntry`, `RulesSnapshotRecord` 7 years; `apiusagedaily` 90 days), `standards/nfrs-and-slos.md` (add BusinessRules.API to T3), `repos/registry.md` unchanged; update `last_reviewed` on every touched file
- Modify: spec front matter `implemented_in` (phase 1 PR link) once merged

- [ ] **Step 1: Write failing test** `CheckoutSucceedsWhenRulesServiceUnavailable`: the snapshot source throws and the cache is at v0; quote + v2 create → 200, total = catalog prices (spec AC *BusinessRules.API is down*).
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests`. Expected: PASS without code changes, which confirms Tasks 3 and 12. If it fails, fix in the owning task's files.
- [ ] **Step 3:** Manual check: `aspire run`, stop `businessrules-api` in the dashboard, place an order in WebApp. The order succeeds, and the `rules.snapshot.age_seconds` gauge on ordering-api keeps growing.
- [ ] **Step 4:** Update the docs listed above (cite file paths as sources; per [AGENTS.md](../../AGENTS.md) memory rules).
- [ ] **Step 5: Commit** (eShop) `test(ordering): verify checkout without business rules service`. (eShopOrgMeta) `docs(architecture): record BusinessRules.API foundation`.

### Phase 1 exit checks

- [ ] `dotnet build eShop.Web.slnf` with 0 warnings; `dotnet test --solution eShop.Web.slnf` green; `npm run test:e2e` green.
- [ ] Spectral passes for the Ordering.API and BusinessRules.API OpenAPI documents (every operation declares a success response and a 400/404, and every parameter has a description).
- [ ] ADR-0008 Accepted. Ordering v1 usage visible in `GET api/usage/create-order-v1`.
- [ ] `superpowers:requesting-code-review` → `finishing-a-development-branch`. The PR links the spec and this plan.

---

## Phases 2–5 (outline; plan each in its own file before starting)

Each phase gets `plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments-phase-N.md`,
written with writing-plans against the code as it is after the previous phase merges.

| Phase | Scope (from spec *Rollout*) | Key tasks | Depends on | Spec acceptance criteria covered |
|-------|-----------------------------|-----------|------------|----------------------------------|
| **2 Rule types** | PRC-001, PRM-001, SHP-001, ORD-001 per-order, ORD-013 incl. partial fulfilment | `IRuleType` implementations + validation (ranges in spec table); targeted entries with most-specific-wins (item > type > brand > all); fill calculator steps (best single promotion per level, order-discount allocation capped by ORD-010); `Order.GracePeriodEndsAt` + `StockShortfallPolicy` + `OrderItem.IsUnfulfillable` + `PromotionLabel` (migration + backfill `OrderDate + 1 min`, `AllOrNothing`); OrderProcessor query uses `"GracePeriodEndsAt" <= CURRENT_TIMESTAMP` (`GracePeriodOrdersRepository.cs:20-27`); partial-fulfilment branch in `SetCancelledStatusWhenStockIsRejected` + paid-event excludes unfulfillable lines; WebApp/ClientApp display; e2e "published promotion charged" | Phase 1 | T-shirt 10% off; future `EffectiveFrom`; partial fulfilment; ORD-001 = 10 min; J5 green |
| **3 eShopAdmin** | Onboard repo; rules UI (J8) | [repo-onboarding-checklist.md](../../templates/repo-onboarding-checklist.md); registry row, workspace folder, `specs/eShopAdmin/`; Blazor + WebAppComponents; list/edit/schedule/publish/history; guard override UX; WCAG 2.2 AA; Playwright "publish a rule" | Phase 2 (rule types to edit) | J8 real; RulesEditor 403 in UI |
| **4 Experiments** | Model, override lock, assignment, snapshot overrides, exposure, order assignments, `OrderPricedIntegrationEvent` | Experiment aggregate + state machine + immutability + ≤ 10 running; override lock (global vs. target-kind/disjoint ids) → 409; stable 64-bit hash fixtures (±1% over 100k); `ExperimentSubjectId` + `experiment_subject` claim; signed visitor cookie + first-sign-in link; quote takes subject key; `OrderExperimentAssignment`; event via outbox; WebApp subscribes to snapshot event; eShopAdmin experiment UI; new journey J10 | Phases 1–3 | E1+E2 independent; anonymous → signed-in sticky; conflict 409; stop → control |
| **5 Tracking and results** | Page view/click/exposure/identify tracking, ingest, aggregation, dashboards | Consent flag (opt-in until legal sign-off); `data-track` attribute; `POST api/tracking/events` (≤ 50 events, ≤ 32 KB, rate limits, HMAC subject with Key Vault secret); day-partitioned `TrackingEvent` (90 d); outcomes from order events (skip `TrackingConsent = false`); 15-min aggregation; z-test/Welch CIs, "not enough data" < 100; erasure by subject HMAC; alerts | Phase 4; **legal sign-off on consent** | Non-consent excluded; results with CIs; erasure |

## Self-review notes

- **Spec coverage (phase 1 rollout item → task):**
  - BusinessRules.API: types → T6; versions → T6; publish → T8; audit → T6, T8, T10; snapshot → T7–T8
  - Shared lib → T2–T3; `rulesdb` and AppHost → T5; Identity → T4
  - Ordering server pricing + Catalog client → T12; quote → T12; v2 → T13; `GetTotal` → T11
  - WebApp → T15; ClientApp → T16; v1 metric + guard → T9–T10; ADR-0008 → T1
  - Resilience AC → T17; RulesEditor-403 AC → T8; tampered-price AC → T13; price-changed AC → T13 + T15/T16
- Glossary names used as in [domain-glossary.md](../../architecture/domain-glossary.md): *rule type, rule version, rules snapshot, price quote*.
- Type names are consistent across tasks: `PriceQuote`, `PricedLine`, `QuoteLine`, `RulesSnapshot`, `RulesSnapshotCache`, `IRulesSnapshotSource`, `IOrderPricingService`, `IApiUsageRecorder`, `IPublishGuard`.
