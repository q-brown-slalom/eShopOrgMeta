---
title: "Runtime business rules and A/B experiments — phase 2 (rule types) implementation plan"
repo: "cross-repo"
status: Draft
approved_by: []
owner: "Q Brown"
created: 2026-11-03
last_reviewed: 2026-11-03
spec: "specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md"
scope: "Phase 2 (Rule types) from the spec's Rollout section. Builds on the phase 1 code as merged (plans/cross-repo/2026-11-03-runtime-business-rules-and-experiments.md)."
---

# Runtime Business Rules — Phase 2 (Rule Types) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the five production rule types: PRC-001 price adjustment, PRM-001 promotion, SHP-001
shipping fee, ORD-001 per-order grace period, and ORD-013 stock shortfall policy with partial
fulfilment. Business admins can then publish them through the BusinessRules.API admin API (Scalar),
and they take effect without a deployment.

**Architecture:** Rule types are code in `eShop.BusinessRules.Client`. Each one is a typed
`RuleType<TParameters>` that validates its JSON parameters (used by BusinessRules.API when drafting)
and resolves the effective value from a `RulesSnapshot` (used by Ordering.API). The phase 1
`PricingCalculator` fills in its four placeholder steps. It becomes the only place that rounds, and it
allocates order discounts in whole cents. Ordering stores the grace period end and shortfall policy on
each order at creation. On stock rejection, an order with `PartialFulfilment` marks the rejected lines
as unfulfillable and reprices downward instead of cancelling. OrderProcessor reads each order's own
`GracePeriodEndsAt`.

**Tech Stack:** .NET 10, Aspire 13.6, Minimal APIs, EF Core + Npgsql, System.Text.Json, RabbitMQ +
`IntegrationEventLogEF`, Blazor (WebApp), .NET MAUI (ClientApp), MSTest + NSubstitute, xUnit (existing
`Ordering.FunctionalTests` only), Playwright.

**Spec:** [specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md](../../specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md)
(sections *Core model*, *Rule types in scope*, *Pricing*, *Fulfilment capabilities*, *Rollout* item 2).
Phase 1 plan: [2026-11-03-runtime-business-rules-and-experiments.md](2026-11-03-runtime-business-rules-and-experiments.md).

## Prerequisites (Definition of Ready)

- [x] Spec `status` is `Approved` (Q Brown, 2026-11-03).
- [ ] Reviewers accept the **Plan decisions** below (P2-D1 … P2-D10).
- [ ] Phase 1 merged to `main`, or the phase 2 branch starts from the phase 1 branch head
      (`eShop` commit `444386e` or later).
- [ ] Worktree/branch `feature/runtime-business-rules-phase-2` in `eShop` (and
      `docs/runtime-business-rules-phase-2` in eShopOrgMeta), created with `using-git-worktrees`.

## Global Constraints

- No new third-party package. Packages only via `eShop/Directory.Packages.props` (spec *Constraints*).
- Rule *types* are code; rule *values* are data. No expressions or scripts in parameters (spec anti-goal).
- Rule IDs, names, and ranges exactly as the spec table:
  - `PRC-001` "Price adjustment": target item/type/brand/all; percent off **0–90 %** or fixed unit price **> 0**
  - `PRM-001` "Promotion": line (target, percent or amount off, optional min units) or order (amount/percent off when subtotal ≥ X); label **≤ 60 chars, no HTML**
  - `SHP-001` "Shipping fee": fee **0–100**, optional free-shipping threshold **≥ 0**
  - `ORD-001` "Grace period": **1–60 minutes**, default **1**
  - `ORD-013` "Stock shortfall policy": `AllOrNothing` | `PartialFulfilment`, default `AllOrNothing`
- Built-in defaults = today's behaviour (catalog price, no discount, shipping 0, 1 minute, `AllOrNothing`). With no published version, nothing visible changes.
- Targeted rules: **most specific target wins**, item > type > brand > all (spec *Core model*).
- Evaluation order: PRC-001 → best single line promotion per line → best single order promotion → SHP-001 on the discounted subtotal. No stacking within a level.
- Rounding: 2 decimals, `MidpointRounding.AwayFromZero`, per line.
- Invariants: final unit price ≥ 0.01; `OrderItem.Discount` ≤ line total (ORD-010); order total ≥ 0; partial fulfilment reprices **only downward**; unfulfillable lines never become fulfillable again.
- An order's pricing, grace period, and shortfall policy are fixed at creation.
- Ordering.API never calls BusinessRules.API on the request path. It resolves rules only from `RulesSnapshotCache.Current`.
- `OrderStatusChangedToPaidIntegrationEvent.OrderStockItems` excludes unfulfillable lines. No new `OrderStatus` value.
- New order columns follow the ADR-0006 order tiers. They are financial, so the archived tier keeps them.
- Commits: Conventional Commits with service scope, e.g. `feat(ordering): …`. Warnings as errors.

## Plan decisions (review before execution)

These resolve items the spec leaves open. Each is a **(call)** for reviewers.

| # | Decision | Why |
|---|----------|-----|
| P2-D1 | **Parameter JSON** is camelCase (`JsonSerializerDefaults.Web`). Enums are strings only (`JsonStringEnumConverter(allowIntegerValues: false)`), and unknown members are rejected (`JsonUnmappedMemberHandling.Disallow`). Money values allow at most 2 decimal places. Shapes are in Tasks 1–2. | Typos in admin input fail at draft time instead of silently falling back to defaults, and a published price always equals a charged price without extra rounding. |
| P2-D2 | **PRM-001 line selection:** an entry is *applicable* if its target matches and `minUnits` (if set) ≤ units. Among applicable entries, keep the most specific target. Among those, the highest discount wins; on a tie, the first listed. PRC-001 rejects two entries with the same target. PRM-001 allows it, so tiers are possible ("3+ units: 15 %"). | Reconciles the spec's "most specific wins" (targeted rules) with "best single line promotion" (ORD-009). Falling back to a less specific entry when tier units aren't met avoids a surprise "no discount". |
| P2-D3 | PRM-001 line `AmountOff` is **per unit**, capped at the line total. Order `PercentOff`/`AmountOff` is capped at the subtotal after line promotions. Order `minSubtotal` is compared with the subtotal **after line promotions**. | The spec doesn't say. Per unit matches how shoppers read "$5 off". Capping keeps totals ≥ 0. |
| P2-D4 | **Rounding moves into `PricingCalculator`.** The unit price is rounded right after PRC-001 and clamped to ≥ 0.01. Line discounts are computed on the rounded unit price, then rounded and capped. The order discount is allocated in whole cents by largest remainder (`DiscountAllocation.Allocate`). `OrderPricingService.RoundToCents` is deleted. | Phase 1 rounded unit price and discount separately after the calculator. With real discounts, `Round(d)` can exceed `units × Round(p)` by a cent (e.g. 3 × 0.333), which throws ORD-010 in `OrderItem` and returns 500. |
| P2-D5 | Each `OrderItem` stores `OrderDiscountShare`, the part of `Discount` that came from the order promotion. Partial fulfilment keeps the **quoted order-discount amount**, re-allocates it over the remaining lines, and caps it at their remaining line totals. Percent order promotions are not recomputed. | The spec says "promotions as quoted; order-level discount re-allocated … capped by ORD-010". Without the stored share, line and order discounts can't be separated. The cap proves the total only goes down (see Task 6). |
| P2-D6 | **ORD-001 and ORD-013 are resolved in `CreateOrderCommandHandler`** from `RulesSnapshotCache.Current` at handling time, not carried on `PriceQuote`. | They aren't prices. The spec says "resolved at creation". In phase 2, a snapshot changing between the quote and the command only affects these two non-price values. |
| P2-D7 | OrderProcessor drops `BackgroundTaskOptions:GracePeriodTime`. The 1-minute default now lives in `GracePeriodRuleType`, and the backfill covers existing rows. `CheckUpdateTime` (ORD-002) stays. | One source of truth. The spec says the query "uses that column instead of the global setting". |
| P2-D8 | Order summaries gain `DroppedItems` (product names of unfulfillable lines). `/user/orders` shows them under the order row with the copy **"Removed (out of stock): {names}"**. The order `Description` copy is **"These items were out of stock and were removed from your order: {names}."** | The spec AC requires the shopper to see the dropped item on `/user/orders`, which today lists summaries only. |
| P2-D9 | **E2E publishes rules for real.** A Playwright helper gets an admin token through the existing implicit `businessrulesswaggerui` client, logged in as `USERNAME1` (alice, `RulesPublisher`, phase 1 D8). It publishes with `guardOverrideReason: "e2e"` and restores each rule's default in `afterAll`. | Covers "published promotion visible and charged" (spec test strategy) before eShopAdmin exists. The guard override is audited, and v1 traffic may exist in a shared dev environment. |
| P2-D10 | Domain enum `Ordering.Domain…StockShortfallPolicy` and shared-lib enum `eShop.BusinessRules.Client.Rules.Types.ShortfallPolicy` have the same members. Ordering.API maps between them. | Ordering.Domain must not reference the client library (hosting dependencies), and two enums with the same name would collide in Ordering.API files. |

## Review Focus

Spec-implied inputs that no task's main tests exercise and that are most likely to bite first. Each
has a test in the task named.

1. **A snapshot rule whose parameters don't parse or validate for this consumer** (schema drift between BusinessRules.API and Ordering.API versions, or a hand-edited row). Quotes and orders must use the default for that rule and keep working, never 500. → Task 4, `UnparseableParametersFallBackToDefault`; Task 1, `ResolveFallsBackToDefaultWhenParametersAreInvalid`.
2. **Cent-level rounding at the ORD-010 boundary** (fractional per-unit prices, 100 % discounts, order discounts split over many cheap lines). Every line must keep `Discount ≤ Units × UnitPrice` after rounding, so `OrderItem` never throws. → Task 5, `OrderDiscountAllocationKeepsEveryLineWithinItsTotal`; Task 4, `PriceAdjustmentNeverGoesBelowOneCent`.
3. **A redelivered or late `OrderStockRejectedIntegrationEvent`** after partial fulfilment was already applied. Expect no second repricing and no status change. → Task 6, `StockRejectionAfterPartialFulfilmentIsIgnored`.
4. **A stock-rejected event that names product ids not on the order.** Expect them to be ignored. Under `PartialFulfilment` with no matching lines, the order proceeds as stock-confirmed. → Task 6, `PartialFulfilmentIgnoresRejectedIdsNotOnOrder`.
5. **A discount larger than what it applies to** (line `amountOff` 50 on a 10.00 item; order `amountOff` above the subtotal). Expect the discount capped at the line or subtotal, total = shipping only, no 500. → Task 5, `LineAmountOffIsCappedAtLineTotal` and `OrderAmountOffIsCappedAtSubtotal`.

---

## File map (eShop unless noted)

| Path | Responsibility |
|------|----------------|
| `src/eShop.BusinessRules.Client/Rules/{RuleType,RuleTarget,BusinessRuleTypesExtensions}.cs` (new) | Typed rule-type base (parse, validate, resolve), target matching, registration of all five |
| `src/eShop.BusinessRules.Client/Rules/Types/*.cs` (new) | `GracePeriodRuleType`, `StockShortfallPolicyRuleType`, `ShippingFeeRuleType`, `PriceAdjustmentRuleType`, `PromotionRuleType` + parameter records |
| `src/eShop.BusinessRules.Client/Pricing/{PricingCalculator,DiscountAllocation,PricedLine,PriceQuote}.cs` | Evaluation steps, rounding, cent allocation |
| `src/BusinessRules.API/Extensions/Extensions.cs` | Register the rule types |
| `src/Ordering.Domain/AggregatesModel/OrderAggregate/{Order,OrderItem,StockShortfallPolicy}.cs` | Fulfilment rules, unfulfillable lines, partial fulfilment, totals |
| `src/Ordering.Infrastructure/` | EF config + migration `AddOrderFulfilmentRules` (+ backfill) |
| `src/Ordering.API/Application/{Rules,Commands,Models,Queries,Pricing}/` | Rules resolver, create/stock-rejected handlers, DTOs, read models |
| `src/OrderProcessor/` | Per-order grace period query |
| `src/WebApp/` | Promotion labels, dropped items on `/user/orders` |
| `src/ClientApp/` | Unfulfillable lines on order detail |
| `e2e/` | Rules admin helper, promotion and partial-fulfilment journeys |
| eShopOrgMeta `architecture/*`, `standards/data-retention.md` | Docs |

---

### Task 1: Rule type base + global rule types (ORD-001, ORD-013, SHP-001)

**Files:**
- Create: `src/eShop.BusinessRules.Client/Rules/RuleType.cs`, `Rules/BusinessRuleTypesExtensions.cs`
- Create: `src/eShop.BusinessRules.Client/Rules/Types/{GracePeriodRuleType,StockShortfallPolicyRuleType,ShippingFeeRuleType}.cs` (each file holds its parameter record and any enum)
- Modify: `src/eShop.BusinessRules.Client/BusinessRulesClientExtensions.cs` (`AddBusinessRulesClient` calls `services.AddBusinessRuleTypes()`)
- Test: `tests/BusinessRules.UnitTests/GlobalRuleTypesTests.cs`

**Interfaces:**
- Consumes: `IRuleType`, `RuleTargeting`, `RuleScopes.Default`, `RulesSnapshot.FindEffective`, `AddRuleType<T>()` (phase 1).
- Produces:
  - `public abstract class RuleType<TParameters> : IRuleType where TParameters : class`
    - `protected static JsonSerializerOptions JsonOptions` (P2-D1)
    - `public abstract TParameters DefaultParameters { get; }`. `Default` = `JsonSerializer.SerializeToElement(DefaultParameters, JsonOptions)`.
    - `public IReadOnlyList<string> Validate(JsonElement parameters)`: a deserialize failure returns one error, `"Parameters don't match the {Id} schema: {message}"`. Otherwise it returns `ValidateParameters(parsed)`.
    - `protected abstract IEnumerable<string> ValidateParameters(TParameters parameters)`
    - `public bool TryParse(JsonElement parameters, [NotNullWhen(true)] out TParameters? value)`: true only if it deserializes **and** has no validation errors
    - `public TParameters Resolve(RulesSnapshot snapshot, DateTimeOffset now)`: `FindEffective(Id, RuleScopes.Default, now)`. Uses `TryParse`; falls back to `DefaultParameters` when nothing is effective or parsing fails. Never throws.
    - `protected static bool HasAtMostTwoDecimals(decimal value)`
  - Namespace `eShop.BusinessRules.Client.Rules.Types`:
    - `GracePeriodRuleType : RuleType<GracePeriodParameters>`: `Id "ORD-001"`, `Name "Grace period"`, `AffectsPrice false`, `Global`. `record GracePeriodParameters(int Minutes)`, default `1`, valid `1..60`. Constants `MinMinutes = 1`, `MaxMinutes = 60`.
    - `StockShortfallPolicyRuleType : RuleType<StockShortfallPolicyParameters>`: `Id "ORD-013"`, `Name "Stock shortfall policy"`, `AffectsPrice false`, `Global`. `enum ShortfallPolicy { AllOrNothing, PartialFulfilment }`, `record StockShortfallPolicyParameters(ShortfallPolicy Policy)`, default `AllOrNothing`.
    - `ShippingFeeRuleType : RuleType<ShippingFeeParameters>`: `Id "SHP-001"`, `Name "Shipping fee"`, `AffectsPrice true`, `Global`. `record ShippingFeeParameters(decimal Fee, decimal? FreeShippingThreshold)`, default `(0, null)`. Valid: `0 ≤ Fee ≤ 100`, `FreeShippingThreshold ≥ 0`, both ≤ 2 decimals.
    - Each `ParameterSchemaJson` is a JSON Schema (draft 2020-12) object describing exactly these properties and ranges.
  - `public static IServiceCollection AddBusinessRuleTypes(this IServiceCollection services)`: `AddRuleType<T>()` for every type, plus `TryAddSingleton<T>()` for each concrete type so consumers can inject them. Task 2 adds the two targeted types to this method.

- [ ] **Step 1: Write the failing tests**

```csharp
static JsonElement J(string json) => JsonDocument.Parse(json).RootElement.Clone();

[TestMethod] public void GracePeriodDefaultIsOneMinute()
    => Assert.AreEqual(1, new GracePeriodRuleType().Resolve(RulesSnapshot.Empty, Now).Minutes);

[TestMethod] [DataRow("""{"minutes":0}""")] [DataRow("""{"minutes":61}""")] [DataRow("""{}""")]
public void GracePeriodOutsideOneToSixtyIsInvalid(string json)
    => Assert.IsNotEmpty(new GracePeriodRuleType().Validate(J(json)));

[TestMethod] public void GracePeriodOfSixtyIsValid() => Assert.IsEmpty(new GracePeriodRuleType().Validate(J("""{"minutes":60}""")));

[TestMethod] public void UnknownMemberIsInvalid()          // P2-D1
    => Assert.IsNotEmpty(new GracePeriodRuleType().Validate(J("""{"minutes":5,"minuets":5}""")));

[TestMethod] public void ShortfallPolicyRejectsIntegerEnum()
    => Assert.IsNotEmpty(new StockShortfallPolicyRuleType().Validate(J("""{"policy":1}""")));

[TestMethod] public void ShortfallPolicyResolvesPublishedValue()
    => Assert.AreEqual(ShortfallPolicy.PartialFulfilment, new StockShortfallPolicyRuleType()
        .Resolve(Snapshot(Rule("ORD-013", """{"policy":"PartialFulfilment"}""")), Now).Policy);

[TestMethod] [DataRow("""{"fee":100.01}""")] [DataRow("""{"fee":-1}""")] [DataRow("""{"fee":4.999}""")]
[DataRow("""{"fee":5,"freeShippingThreshold":-0.01}""")]
public void ShippingFeeOutOfRangeIsInvalid(string json) => Assert.IsNotEmpty(new ShippingFeeRuleType().Validate(J(json)));

[TestMethod] public void ResolveFallsBackToDefaultWhenParametersAreInvalid()   // Review Focus 1
    => Assert.AreEqual(1, new GracePeriodRuleType().Resolve(Snapshot(Rule("ORD-001", """{"minutes":"ten"}""")), Now).Minutes);

[TestMethod] public void DefaultSerializesCamelCase()
    => Assert.AreEqual(0m, new ShippingFeeRuleType().Default.GetProperty("fee").GetDecimal());

[TestMethod] public void AddBusinessRuleTypesRegistersGlobalTypes()
{   var registry = new ServiceCollection().AddBusinessRuleTypes().BuildServiceProvider().GetRequiredService<RuleTypeRegistry>();
    Assert.IsNotNull(registry.Find("ORD-001")); Assert.IsNotNull(registry.Find("ORD-013")); Assert.IsNotNull(registry.Find("SHP-001")); }
```

`Snapshot(params SnapshotRuleVersion[])` and `Rule(string typeId, string json, int version = 1, int fromHours = -1)` are test helpers. Put them in a new `tests/BusinessRules.UnitTests/SnapshotFactory.cs` so Tasks 2, 4, and 5 can reuse them.

- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests`. Expected: build FAIL (types missing).
- [ ] **Step 3: Implement** the members listed under Interfaces.
- [ ] **Step 4: Run** the same command. Expected: PASS, and the phase 1 tests stay green.
- [ ] **Step 5: Commit** `feat(businessrules): add grace period, shortfall policy and shipping fee rule types`

### Task 2: Targeted rule types (PRC-001, PRM-001)

**Files:**
- Create: `src/eShop.BusinessRules.Client/Rules/RuleTarget.cs`
- Create: `src/eShop.BusinessRules.Client/Rules/Types/{PriceAdjustmentRuleType,PromotionRuleType}.cs`
- Modify: `src/eShop.BusinessRules.Client/Rules/BusinessRuleTypesExtensions.cs`
- Test: `tests/BusinessRules.UnitTests/TargetedRuleTypesTests.cs`

**Interfaces:**
- Consumes: `RuleType<T>`, `SnapshotFactory` (Task 1); `CatalogPrice` (phase 1).
- Produces:
  - `public enum RuleTargetKind { All, Brand, Type, Item }`. The numeric order is the specificity order.
  - `public sealed record RuleTarget(RuleTargetKind Kind, int? Id) { bool Matches(CatalogPrice product); int Specificity => (int)Kind; }`. `All` requires `Id == null`. The others require `Id > 0` and match `CatalogBrandId` / `CatalogTypeId` / `ProductId`.
  - `PriceAdjustmentRuleType : RuleType<PriceAdjustmentParameters>`: `Id "PRC-001"`, `Name "Price adjustment"`, `AffectsPrice true`, `Targeted`.
    - `enum PriceAdjustmentMode { PercentOff, FixedPrice }`
    - `record PriceAdjustmentEntry(RuleTarget Target, PriceAdjustmentMode Mode, decimal Value)`
    - `record PriceAdjustmentParameters(IReadOnlyList<PriceAdjustmentEntry> Entries)`, default `[]`
    - Valid: `entries` required and ≤ 500; `PercentOff` in `0..90`; `FixedPrice` > 0 with ≤ 2 decimals; no two entries share `(Kind, Id)`.
  - `PromotionRuleType : RuleType<PromotionParameters>`: `Id "PRM-001"`, `Name "Promotion"`, `AffectsPrice true`, `Targeted`.
    - `enum DiscountKind { PercentOff, AmountOff }`
    - `record LinePromotion(RuleTarget Target, DiscountKind Kind, decimal Value, int? MinUnits, string Label)`
    - `record OrderPromotion(DiscountKind Kind, decimal Value, decimal MinSubtotal, string Label)`
    - `record PromotionParameters(IReadOnlyList<LinePromotion> LinePromotions, IReadOnlyList<OrderPromotion> OrderPromotions)`, default both `[]`
    - Valid: both lists required and ≤ 500 each; `PercentOff` in `(0, 100]`; `AmountOff` > 0 with ≤ 2 decimals; `MinUnits` null or ≥ 1; `MinSubtotal` ≥ 0 with ≤ 2 decimals; `Label` trimmed length 1–60 with no `<` or `>`.
  - Error messages carry a JSON path, e.g. `"entries[2].value: percent off must be between 0 and 90."`.
  - Example published parameters (also used by Tasks 7 and 10):
    `{"entries":[{"target":{"kind":"Type","id":1},"mode":"PercentOff","value":10}]}` and
    `{"linePromotions":[{"target":{"kind":"Item","id":99},"kind":"PercentOff","value":10,"minUnits":null,"label":"10% off"}],"orderPromotions":[]}`

- [ ] **Step 1: Write the failing tests**
  - `TargetMatchesByKind` (DataRows: All matches anything; Brand 2 matches a product with brand 2 only; Type and Item likewise)
  - `PriceAdjustmentPercentOffAbove90IsInvalid`, `PriceAdjustmentFixedPriceZeroIsInvalid`, `PriceAdjustmentDuplicateTargetIsInvalid`, `TargetAllWithIdIsInvalid`, `TargetItemWithoutIdIsInvalid`
  - `PromotionLabelLongerThan60IsInvalid`, `PromotionLabelWithHtmlIsInvalid` (`"<b>Sale</b>"`), `PromotionPercentOffZeroIsInvalid`, `PromotionMinUnitsZeroIsInvalid`, `PromotionMissingOrderPromotionsIsInvalid`
  - `PromotionDuplicateTargetsAreValid` (two tiers on item 7; P2-D2)
  - `ExampleParametersAreValid` (both example documents above → no errors)
  - `AddBusinessRuleTypesRegistersAllFive`
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): add price adjustment and promotion rule types`

### Task 3: BusinessRules.API registers the rule types

**Files:**
- Modify: `src/BusinessRules.API/Extensions/Extensions.cs`: replace the phase 1 comment and `TryAddSingleton<RuleTypeRegistry>()` with `services.AddBusinessRuleTypes()`
- Test: `tests/BusinessRules.FunctionalTests/RuleTypesApiTests.cs`

**Interfaces:**
- Consumes: `AddBusinessRuleTypes()` (Tasks 1–2); phase 1 endpoints `ListRuleTypes`, `CreateRuleDraft`, `PublishRuleVersion`; `FakeOrderingUsageClient`.
- Produces: the five types in `GET api/rules/types`. Drafts are validated by the type-specific rules (phase 1 `CreateDraftRequestValidator` already calls `IRuleType.Validate`).

- [ ] **Step 1: Write failing tests**
  - `ListRuleTypesReturnsFiveProductionTypes`: ids contain `PRC-001, PRM-001, SHP-001, ORD-001, ORD-013`. `affectsPrice` is true for exactly the first three. `ORD-001` default is `{"minutes":1}`.
  - `CreateDraftWithGracePeriodOf61MinutesReturnsBadRequest`: 400, `errors["Parameters"]` not empty
  - `CreateDraftWithHtmlPromotionLabelReturnsBadRequest`
  - `PublishShippingFeeIsGuardedWhileV1CallsExist` (fake usage client reports 3 calls → 409; this shows `AffectsPrice` reaches the guard)
  - `PublishGracePeriodIsNotGuarded` (same fake → 200)
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(businessrules): register production rule types`

### Task 4: Calculator — PRC-001 price adjustments and rounding in the calculator

**Files:**
- Modify: `src/eShop.BusinessRules.Client/Pricing/PricingCalculator.cs` (`ApplyPriceAdjustments` and `CreateBaseLine`, ≈ lines 85–105)
- Modify: `src/Ordering.API/Application/Pricing/OrderPricingService.cs` (delete `RoundToCents`; return the calculator's quote, P2-D4)
- Test: `tests/BusinessRules.UnitTests/PricingCalculatorTests.cs`, `tests/Ordering.UnitTests/Application/OrderPricingServiceTests.cs`

**Interfaces:**
- Consumes: `PriceAdjustmentRuleType.Resolve` and `RuleTarget.Matches` (Tasks 1–2).
- Produces:
  - `PricingCalculator` keeps its public signature and parameterless constructor. It holds private static instances of the three price rule types.
  - After `ApplyPriceAdjustments`: `BasePrice = Round(catalog price)`; `UnitPrice = Max(0.01m, Round(adjusted))`, where adjusted = `BasePrice × (1 − value/100)` for `PercentOff` or `value` for `FixedPrice`, from the single most specific matching entry. No match → `UnitPrice = BasePrice`.
  - `IOrderPricingService` behaviour is unchanged apart from rounding now coming from the calculator.

- [ ] **Step 1: Write failing tests** (catalog helper gains type and brand: `Catalog((id, price, typeId, brandId), …)`)
  - `PriceAdjustmentPercentOffByTypeReducesUnitPrice`: type 2, 10 % off 79.99 → `UnitPrice` 71.99, `BasePrice` 79.99, `Total` 71.99
  - `PriceAdjustmentMostSpecificTargetWins`: `All` 10 % + `Item` fixed 5.00 → 5.00; `Brand` 20 % + `Type` 10 % on 100.00 → 90.00
  - `PriceAdjustmentNeverGoesBelowOneCent`: base 0.01, 90 % off → 0.01 (Review Focus 2)
  - `UnitPriceIsRoundedHalfAwayFromZero`: base 10.05, 50 % off → 5.03
  - `PriceAdjustmentTakesEffectAtEffectiveFrom`: a version from `Now + 1h` is ignored at `Now` and applied at `Now + 1h` (spec AC *future `EffectiveFrom`*)
  - `UnparseableParametersFallBackToDefault`: PRC-001 parameters `{"entries":"oops"}` → catalog price, no exception (Review Focus 1)
  - Ordering unit: `QuoteReturnsCalculatorRoundingUnchanged` (replaces any `RoundToCents` tests: substitute snapshot with a 33 % off rule; the service returns the same `UnitPrice` as the calculator)
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests` and `dotnet test --project tests/Ordering.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both, plus `dotnet test --project tests/Ordering.FunctionalTests --filter "FullyQualifiedName~PricingApiTests"`. Expected: PASS (no-rule quotes unchanged).
- [ ] **Step 5: Commit** `feat(businessrules): apply price adjustments and round in the pricing calculator`

### Task 5: Calculator — promotions, order discount allocation, shipping

**Files:**
- Create: `src/eShop.BusinessRules.Client/Pricing/DiscountAllocation.cs`
- Modify: `src/eShop.BusinessRules.Client/Pricing/{PricingCalculator,PricedLine,PriceQuote}.cs`
- Test: `tests/BusinessRules.UnitTests/{PricingCalculatorTests,DiscountAllocationTests}.cs`

**Interfaces:**
- Consumes: `PromotionRuleType`, `ShippingFeeRuleType` (Tasks 1–2); rounded lines (Task 4).
- Produces:
  - `public static class DiscountAllocation { public static decimal[] Allocate(decimal amount, IReadOnlyList<decimal> capacities); }`
    - Throws `ArgumentOutOfRangeException` if `amount < 0`, `amount > Σ capacities`, or `amount` has more than 2 decimals.
    - Each share is proportional to its capacity, floored to the cent. The leftover cents go one at a time by largest remainder; ties go to the lower index.
    - Guarantees `Σ result == amount` and `0 ≤ result[i] ≤ capacities[i]`. Zero capacity → 0. Task 6 reuses it.
  - `PricedLine` gains a trailing `decimal OrderDiscountShare = 0m` (included in `Discount`). `PriceQuote` gains a trailing `string? OrderPromotionLabel = null` and a computed `decimal OrderDiscount => Lines.Sum(l => l.OrderDiscountShare)`. Both are additive, so existing callers compile.
  - Steps (P2-D2, P2-D3):
    - **Line promotions** pick one entry per line. `PercentOff` → `Round(Units × UnitPrice × v/100)`; `AmountOff` → `Round(Units × v)`. Capped at `Units × UnitPrice`. Sets `Discount` and `PromotionLabel`.
    - **Order promotion:** let `s = Σ LineTotal` after line promotions. Applicable if `s ≥ MinSubtotal`. The amount is `Round(s × v/100)` or `v`, capped at `s`. The highest amount wins (first listed on a tie). It is allocated over line totals with `DiscountAllocation.Allocate`, each share is added to `Discount` and `OrderDiscountShare`, and `OrderPromotionLabel` is set.
    - **Shipping:** let `d = Subtotal − DiscountTotal`. `ShippingFee = (threshold is not null && d ≥ threshold) ? 0 : Fee`.

- [ ] **Step 1: Write failing tests**

```csharp
[TestMethod] public void DiscountAllocationDistributesRemainderCentsByLargestRemainder()
    => CollectionAssert.AreEqual(new[] { 0.04m, 0.03m, 0.03m }, DiscountAllocation.Allocate(0.10m, [1m, 1m, 1m]));

[TestMethod] public void BestOrderPromotionIsAllocatedProportionallyInCents()
{   // lines 30.00 + 10.00; order promotions: 10.00 off at ≥ 40, 5 % off at ≥ 0  → 10.00 wins
    var q = Calc([new(1, 1), new(2, 1)], Catalog((1, 30m), (2, 10m)), Snapshot(Rule("PRM-001", Promo(order: [Amount(10m, min: 40m, "Spend 40 save 10"), Pct(5m, min: 0m, "5%")]))));
    Assert.AreEqual(7.50m, q.Lines[0].OrderDiscountShare); Assert.AreEqual(2.50m, q.Lines[1].OrderDiscountShare);
    Assert.AreEqual(30.00m, q.Total); Assert.AreEqual("Spend 40 save 10", q.OrderPromotionLabel); }

[TestMethod] public void OrderDiscountAllocationKeepsEveryLineWithinItsTotal()   // Review Focus 2
{   // 3 × 0.33, 1 × 0.01, 2 × 0.05; 100 % order discount
    var q = Calc([new(1, 3), new(2, 1), new(3, 2)], Catalog((1, 0.33m), (2, 0.01m), (3, 0.05m)), Snapshot(Rule("PRM-001", Promo(order: [Pct(100m, min: 0m, "Free")]))));
    Assert.IsTrue(q.Lines.All(l => l.Discount <= l.Units * l.UnitPrice)); Assert.AreEqual(0m, q.Total); }
```

  - `LinePercentPromotionSetsDiscountAndLabel`: 2 × 50.00, 10 % → `Discount` 10.00, label set, `Total` 90.00
  - `LineAmountOffIsPerUnit`: 3 × 20.00, 5 off → 15.00
  - `LineAmountOffIsCappedAtLineTotal`: 1 × 10.00, 50 off → `Discount` 10.00, `Total` 0 (Review Focus 5)
  - `OrderAmountOffIsCappedAtSubtotal`: subtotal 12.00, 20 off → `DiscountTotal` 12.00 (Review Focus 5)
  - `LinePromotionMinUnitsNotMetFallsBackToLessSpecificEntry`: item tier 15 % needs 3 units, `All` 5 %; 2 units → 5 %
  - `LinePromotionMostSpecificApplicableWinsThenHighestDiscount`: `All` 20 % vs `Item` 5 % → 5 %; two item tiers (min 1: 5 %, min 3: 15 %) with 3 units → 15 %
  - `LinePromotionAppliesAfterPriceAdjustment`: PRC 10 % off 100.00 → 90.00; PRM 10 % → `Discount` 9.00
  - `OrderPromotionThresholdUsesSubtotalAfterLinePromotions`: subtotal 50.00, line discount 5.00, `minSubtotal` 50 → no order discount
  - `ShippingFeeAppliesBelowThresholdAndIsFreeAtThreshold`: fee 4.99, threshold 50; discounted subtotal 49.99 → 4.99; 50.00 → 0
  - `ShippingWithoutThresholdIsAlwaysCharged`
  - `DiscountAllocationRejectsAmountAboveCapacity`
- [ ] **Step 2: Run** `dotnet test --project tests/BusinessRules.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.** Keep each step a named private method, as phase 1 laid out.
- [ ] **Step 4: Run.** Expected: PASS, and phase 1 calculator tests stay green.
- [ ] **Step 5: Commit** `feat(businessrules): apply promotions, allocate order discounts and charge shipping`

### Task 6: Ordering domain — fulfilment rules, unfulfillable lines, partial fulfilment

**Files:**
- Create: `src/Ordering.Domain/AggregatesModel/OrderAggregate/StockShortfallPolicy.cs`
- Modify: `src/Ordering.Domain/AggregatesModel/OrderAggregate/{Order,OrderItem}.cs`
- Modify: `src/Ordering.Infrastructure/EntityConfigurations/{OrderEntityTypeConfiguration,OrderItemEntityTypeConfiguration}.cs`
- Create: `src/Ordering.Infrastructure/Migrations/<ts>_AddOrderFulfilmentRules.cs` (+ designer, snapshot update)
- Modify: `tests/Ordering.UnitTests/Builders.cs` (`OrderBuilder.AddOne` passes through `promotionLabel` and `orderDiscountShare`; new `WithFulfilmentRules(TimeSpan, StockShortfallPolicy)`)
- Test: `tests/Ordering.UnitTests/Domain/{OrderAggregateTest,PartialFulfilmentTest}.cs`

**Interfaces:**
- Consumes: `DiscountAllocation.Allocate` (Task 5; Ordering.UnitTests reach it through Ordering.API). Ordering.Domain itself takes it only as a delegate.
- Produces:
  - `public enum StockShortfallPolicy { AllOrNothing, PartialFulfilment }`
  - `Order.GracePeriodEndsAt` (`DateTime`, UTC) and `Order.StockShortfallPolicy`. The constructor sets `OrderDate + 1 min` and `AllOrNothing`.
  - `void Order.SetFulfilmentRules(TimeSpan gracePeriod, StockShortfallPolicy policy)`: `GracePeriodEndsAt = OrderDate + gracePeriod`. Throws `OrderingDomainException` unless `OrderStatus == Submitted` and `1 min ≤ gracePeriod ≤ 60 min`.
  - `Order.AddOrderItem(int productId, string productName, decimal unitPrice, decimal discount, string pictureUrl, int units = 1, decimal? basePrice = null, string? promotionLabel = null, decimal orderDiscountShare = 0m)`
  - `OrderItem`: `string? PromotionLabel`, `decimal OrderDiscountShare` (constructor validates `0 ≤ share ≤ discount`), `bool IsUnfulfillable`. Internal `MarkUnfulfillable()` and `ReplaceOrderDiscountShare(decimal share)`, which sets `Discount = Discount − OrderDiscountShare + share`.
  - `void Order.ApplyStockRejection(IReadOnlyCollection<int> rejectedProductIds, Func<decimal, IReadOnlyList<decimal>, IReadOnlyList<decimal>> allocateOrderDiscount)`. It does nothing unless `OrderStatus == AwaitingValidation`.
    - `AllOrNothing` → `SetCancelledStatusWhenStockIsRejected(rejectedProductIds)` (unchanged).
    - `PartialFulfilment`: rejected = lines whose `ProductId` is in the ids. Ids not on the order are ignored.
      - None rejected → `SetStockConfirmedStatus()`.
      - All rejected → cancel as today.
      - Otherwise, let amount = Σ `OrderDiscountShare` over all lines. Mark the rejected lines unfulfillable. Call `allocateOrderDiscount(min(amount, Σ capacities), capacities)` over the remaining lines, where capacity = `Units × UnitPrice − (Discount − OrderDiscountShare)`. Apply the shares with `ReplaceOrderDiscountShare`, recalculate totals, call `SetStockConfirmedStatus()`, then set `Description` to the P2-D8 copy (product names in line order, comma-separated).
  - Totals (`RecalculateTotals`, `ApplyPricing`, `GetTotal`) and the `OrderStatusChangedToPaidDomainEvent` items **exclude** unfulfillable lines. `ShippingFee` is unchanged.
  - EF: `StockShortfallPolicy` `HasConversion<string>().HasMaxLength(30)`; `PromotionLabel` `HasMaxLength(60)`.
  - Migration (`dotnet ef migrations add AddOrderFulfilmentRules --startup-project src/Ordering.API --project src/Ordering.Infrastructure --context OrderingContext`) adds `orders."GracePeriodEndsAt"` (timestamptz), `orders."StockShortfallPolicy"` (default `'AllOrNothing'`), `orderItems."IsUnfulfillable"` (default false), `orderItems."PromotionLabel"` (nullable), and `orderItems."OrderDiscountShare"` (default 0). `Up` backfills with `UPDATE ordering.orders SET "GracePeriodEndsAt" = "OrderDate" + interval '1 minute';`.
  - **Retention:** the new columns follow the ADR-0006 order tiers (financial and non-personal).

- [ ] **Step 1: Write failing tests**

```csharp
[TestMethod] public void PartialFulfilmentMarksRejectedLinesAndRepricesDownward()
{   // lines 10, 20, 30 (×1); order discount 6.00 allocated 1/2/3; shipping 5 → Total 59.00
    var order = new OrderBuilder(new AddressBuilder().Build())
        .AddOne(1, "P1", 10m, 1m, "", orderDiscountShare: 1m).AddOne(2, "P2", 20m, 2m, "", orderDiscountShare: 2m)
        .AddOne(3, "P3", 30m, 3m, "", orderDiscountShare: 3m).WithPricing(5m, 1)
        .WithFulfilmentRules(TimeSpan.FromMinutes(1), StockShortfallPolicy.PartialFulfilment).Build();
    order.SetAwaitingValidationStatus();
    order.ApplyStockRejection([2], DiscountAllocation.Allocate);
    Assert.AreEqual(OrderStatus.StockConfirmed, order.OrderStatus);
    Assert.IsTrue(order.OrderItems.Single(i => i.ProductId == 2).IsUnfulfillable);
    Assert.AreEqual(40m, order.Subtotal); Assert.AreEqual(6m, order.DiscountTotal); Assert.AreEqual(39m, order.Total);
    Assert.AreEqual("These items were out of stock and were removed from your order: P2.", order.Description); }
```

  - `NewOrderDefaultsToOneMinuteGracePeriodAndAllOrNothing`
  - `SetFulfilmentRulesStoresGracePeriodEnd` (10 min → `GracePeriodEndsAt − OrderDate == 10 min`; spec AC ORD-001 = 10)
  - `SetFulfilmentRulesRejectsGracePeriodOutsideOneToSixtyMinutes` (DataRow 0, 61)
  - `AllOrNothingStockRejectionCancelsOrder` (existing behaviour through `ApplyStockRejection`)
  - `PartialFulfilmentCapsReallocatedOrderDiscountAtRemainingLineTotals`: lines 5.00 and 20.00, order discount 15.00 (shares 3/12), reject the 20.00 line → remaining discount 5.00, `Total` = shipping
  - `PartialFulfilmentWithEveryLineRejectedCancels`
  - `StockRejectionAfterPartialFulfilmentIsIgnored`: second call → same totals, status, and description (Review Focus 3)
  - `PartialFulfilmentIgnoresRejectedIdsNotOnOrder`: rejected `[99]` → `StockConfirmed`, nothing unfulfillable (Review Focus 4)
  - `PaidDomainEventExcludesUnfulfillableLines`: after partial fulfilment + `SetPaidStatus()`, the event's items are products 1 and 3
  - `GetTotalExcludesUnfulfillableLines`
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement**, then create the migration and add the backfill SQL to `Up`.
- [ ] **Step 4: Run.** Expected: PASS. Then `dotnet test --project tests/Ordering.FunctionalTests` → existing tests green (migration applies).
- [ ] **Step 5: Commit** `feat(ordering): store fulfilment rules per order and support partial fulfilment`

### Task 7: Ordering.API — resolve rules at creation and persist promotions

**Files:**
- Create: `src/Ordering.API/Application/Rules/{IOrderFulfilmentRulesResolver,OrderFulfilmentRulesResolver,OrderFulfilmentRules}.cs`
- Modify: `src/Ordering.API/Application/Commands/{CreateOrderCommand,CreateOrderCommandHandler}.cs`, `Application/Models/OrderItemDTO.cs`, `Extensions/Extensions.cs` (register the resolver as a singleton)
- Create: `tests/Ordering.FunctionalTests/TestRulesSnapshotSource.cs`. It replaces `UnavailableRulesSnapshotSource.cs` (delete it) and keeps `Calls` and `FirstCallAttempted`.
- Modify: `tests/Ordering.FunctionalTests/OrderingApiFixture.cs`, `ResilienceTests.cs` (type rename only)
- Test: `tests/Ordering.FunctionalTests/RuleTypesOrderingTests.cs` (own `IClassFixture<OrderingApiFixture>`), `tests/Ordering.UnitTests/Application/{NewOrderCommandHandlerTest,OrderFulfilmentRulesResolverTests}.cs`

**Interfaces:**
- Consumes: `GracePeriodRuleType`, `StockShortfallPolicyRuleType`, `ShortfallPolicy` (Task 1); `PricedLine.PromotionLabel` / `OrderDiscountShare` (Task 5); `Order.SetFulfilmentRules`, `StockShortfallPolicy`, extended `AddOrderItem` (Task 6); `RulesSnapshotCache` (phase 1).
- Produces:
  - `sealed record OrderFulfilmentRules(TimeSpan GracePeriod, StockShortfallPolicy ShortfallPolicy)`
  - `interface IOrderFulfilmentRulesResolver { OrderFulfilmentRules Resolve(); }`. The implementation uses `cache.Current`, `TimeProvider.GetUtcNow()`, and both rule types, and maps `ShortfallPolicy` → `StockShortfallPolicy` (P2-D10).
  - `CreateOrderCommandHandler` takes the resolver as a new last constructor parameter and calls `order.SetFulfilmentRules(...)` before `ApplyPricing`. It adds `rules.grace_period_minutes` and `rules.shortfall_policy` to the existing log scope.
  - `OrderItemDTO` gains `string? PromotionLabel` and `decimal OrderDiscountShare`. `CreateOrderCommand(PriceQuote …)` maps them from each line.
  - `TestRulesSnapshotSource : IRulesSnapshotSource`: throws while `Snapshot` is null (so `ResilienceTests` keep passing) and otherwise returns it.
  - `Task OrderingApiFixture.PublishSnapshotAsync(params SnapshotRuleVersion[] rules)`: sets a snapshot with the next version that contains **only** `rules` (it replaces everything earlier tests in the class published), then awaits `RulesSnapshotCache.RefreshAsync`.

- [ ] **Step 1: Write failing tests**
  - Functional `TypeTenPercentOffIsQuotedAndChargedExactly`: publish PRC-001 `Type 1` 10 % (the Task 2 example). Quote 2 × product 1 (10.00, type 1) → `UnitPrice` 9.00, `Total` 18.00. v2 create with `expectedTotal` 18.00 → 200. The stored order has `Total` 18.00 and the item has `BasePrice` 10.00, `UnitPrice` 9.00. (Spec AC *T-shirt 10 % off*, at the API layer.)
  - Functional `PromotionLabelAndShippingAreStoredOnOrder`: PRM-001 line 10 % on item 2 labelled "10% off" + SHP-001 fee 4.99 → `GET api/orders/{id}` shows the item `PromotionLabel` "10% off" and `ShippingFee` 4.99.
  - Functional `OrderStoresResolvedGracePeriodAndShortfallPolicy`: ORD-001 10, ORD-013 `PartialFulfilment` → in `OrderingContext`, `GracePeriodEndsAt − OrderDate == 10 min` and the policy is `PartialFulfilment`.
  - Unit `ResolverReturnsDefaultsForEmptySnapshot` (1 min, `AllOrNothing`), `ResolverMapsPartialFulfilment`
  - Unit `HandlerAppliesResolvedFulfilmentRules` (update `NewOrderCommandHandlerTest` for the new constructor parameter)
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.UnitTests` and `dotnet test --project tests/Ordering.FunctionalTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** both. Expected: PASS, including `ResilienceTests`.
- [ ] **Step 5: Commit** `feat(ordering): resolve grace period and shortfall policy at order creation`

### Task 8: Ordering.API — partial fulfilment on stock rejection and read models

**Files:**
- Modify: `src/Ordering.API/Application/Commands/SetStockRejectedOrderStatusCommandHandler.cs` (call `ApplyStockRejection(command.OrderStockItems, DiscountAllocation.Allocate)` instead of `SetCancelledStatusWhenStockIsRejected`)
- Modify: `src/Ordering.API/Application/Queries/{OrderViewModel,OrderQueries}.cs`
- Test: `tests/Ordering.FunctionalTests/PartialFulfilmentTests.cs` (own fixture)

**Interfaces:**
- Consumes: `Order.ApplyStockRejection` (Task 6); `OrderingApiFixture.PublishSnapshotAsync` (Task 7); the existing `SetAwaitingValidationOrderStatusCommand`, `SetStockRejectedOrderStatusCommand`, and `SetPaidOrderStatusCommand` (sent through `IMediator` from a scope in the test).
- Produces (additive read model changes):
  - `Orderitem`: `decimal BasePrice`, `string? PromotionLabel`, `bool IsUnfulfillable`
  - `OrderSummary`: `List<string> DroppedItems` (names of unfulfillable lines; empty when there are none). Projected in the EF query, not loaded afterwards.
  - The paid integration event needs no handler change: `OrderStatusChangedToPaidDomainEventHandler` already builds `OrderStockItems` from the domain event's items, which Task 6 filters.

- [ ] **Step 1: Write failing tests**
  - `PartialFulfilmentRepricesOrderAndExcludesDroppedLineFromPaidEvent`: publish ORD-013 `PartialFulfilment`, create an order for products 1, 2, 3, then send awaiting-validation, stock-rejected `[2]`, and paid. The order is `Paid` and `Total` = old total − product 2's line total. Product 2 has `isUnfulfillable: true`. The outbox `IntegrationEventLog` row for `OrderStatusChangedToPaidIntegrationEvent` has `OrderStockItems` product ids `[1, 3]`. (Spec AC *one out-of-stock line of three*, Ordering side; the handler's built-in 10 s delay makes this test slow.)
  - `GetOrdersListsDroppedItems`: the same order appears in `GET api/orders` with `droppedItems: ["Fake Product 2"]`
  - `AllOrNothingStockRejectionStillCancels`: no ORD-013 published → `Cancelled`
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests --filter "FullyQualifiedName~PartialFulfilmentTests"`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the full `tests/Ordering.FunctionalTests`. Expected: PASS.
- [ ] **Step 5: Commit** `feat(ordering): partially fulfil orders on stock rejection when the policy allows`

### Task 9: OrderProcessor — per-order grace period

**Files:**
- Modify: `src/OrderProcessor/Services/GracePeriodOrdersRepository.cs`, `Services/GracePeriodManagerService.cs`, `BackgroundTaskOptions.cs` (remove `GracePeriodTime`), `appsettings.json` (remove `GracePeriodTime`)
- Modify: `src/OrderProcessor/OrderProcessor.csproj` (`<InternalsVisibleTo Include="Ordering.FunctionalTests" />`)
- Modify: `tests/Ordering.FunctionalTests/Ordering.FunctionalTests.csproj` (`<ProjectReference Include="..\..\src\OrderProcessor\OrderProcessor.csproj" Aliases="OrderProcessorAssembly" />`; the alias avoids a clash between the two top-level `Program` types)
- Test: `tests/Application.UnitTests/OrderProcessorTests.cs` (update), `tests/Ordering.FunctionalTests/GracePeriodOrdersRepositoryTests.cs` (new, own fixture, `extern alias OrderProcessorAssembly;`)

**Interfaces:**
- Consumes: `orders."GracePeriodEndsAt"` (Task 6), `PublishSnapshotAsync` (Task 7).
- Produces: `IGracePeriodOrdersRepository.GetConfirmedGracePeriodOrdersAsync(CancellationToken cancellationToken) : ValueTask<List<int>>` with the query
  `SELECT "Id" FROM ordering.orders WHERE "GracePeriodEndsAt" <= CURRENT_TIMESTAMP AND "OrderStatus" = 'Submitted'`.

- [ ] **Step 1: Write failing tests**
  - `GracePeriodRepositoryReturnsOnlyOrdersPastTheirOwnGracePeriod`: publish ORD-001 = 10 and create two orders. Set one order's `"GracePeriodEndsAt"` to `now() - interval '1 second'` with SQL, then call the repository (built with `NpgsqlDataSource.Create(fixture connection string)`). Only that order is returned. The other order, 10 minutes out, is not, so it is still cancellable (spec AC ORD-001 = 10 min).
  - Update `PublishesGracePeriodConfirmationForEveryEligibleOrder` for the new signature.
- [ ] **Step 2: Run** `dotnet test --project tests/Ordering.FunctionalTests --filter "FullyQualifiedName~GracePeriodOrdersRepositoryTests"` and `dotnet test --project tests/Application.UnitTests --filter "FullyQualifiedName~OrderProcessorTests"`. Expected: FAIL.
- [ ] **Step 3: Implement.** `src/eShop.AppHost/AppHost.cs:51-54` already makes order-processor `WaitFor(orderingApi)`, which runs migrations, so the backfill lands first (spec *Migration order*). Leave it as is.
- [ ] **Step 4: Run** both. Expected: PASS.
- [ ] **Step 5: Commit** `feat(orderprocessor): use each order's grace period end`

### Task 10: WebApp — promotion labels, dropped items, E2E journeys

**Files:**
- Modify: `src/WebApp/Services/OrderingService.cs` (`OrderRecord` gains trailing `string[]? DroppedItems = null`)
- Modify: `src/WebApp/Components/Pages/User/Orders.razor` (under a row with dropped items: `<p class="order-dropped">Removed (out of stock): @string.Join(", ", order.DroppedItems)</p>`)
- Modify: `src/WebApp/Components/Pages/Cart/CartPage.razor` (≈ line 46: show the quote line's `PromotionLabel` in `<span class="cart-item-promotion">` when present), `Checkout/Checkout.razor` (≈ line 67: the discount `<dt>` reads `Discount (@quote.OrderPromotionLabel)` when a label is present)
- Modify: `src/WebApp/wwwroot/css/` or the components' `.razor.css` for the two new classes (reuse existing tokens)
- Create: `e2e/rules-admin-helpers.ts`, `e2e/PromotionTest.spec.ts`, `e2e/PartialFulfilmentTest.spec.ts`
- Modify: `playwright.config.ts` (add both specs to the `e2e tests logged in` project's `testMatch`), `.env` sample/docs if present (`BUSINESSRULES_URL`, `IDENTITY_URL`)
- Test: `tests/Application.UnitTests/WebAppPricingTests.cs` (add `OrderRecordDeserializesDroppedItems`)

**Interfaces:**
- Consumes: `PriceQuote.OrderPromotionLabel`, `PricedLine.PromotionLabel` (Task 5); `OrderSummary.DroppedItems` (Task 8); BusinessRules.API admin endpoints `GET api/rules/types`, `POST api/rules/versions`, `POST api/rules/versions/{id}/publish` (`api-version=1.0`, phase 1).
- Produces (`e2e/rules-admin-helpers.ts`, P2-D9):
  - `getRulesAdminToken(browser: Browser): Promise<string>`: opens `{IDENTITY_URL}/connect/authorize?client_id=businessrulesswaggerui&response_type=token&scope=businessrules.admin&redirect_uri={BUSINESSRULES_URL}/swagger/oauth2-redirect.html&state=e2e&nonce=e2e`, logs in with `USERNAME1`/`PASSWORD`, and waits for a URL with `access_token=`. Defaults: `IDENTITY_URL` `http://localhost:5223`, `BUSINESSRULES_URL` `http://localhost:5230`.
  - `publishRule(request: APIRequestContext, token: string, ruleTypeId: string, parameters: unknown): Promise<void>`: creates a draft (scope `default`, `effectiveFrom` = now), then publishes it with `{ guardOverrideReason: "e2e" }`. Expects 2xx at each step.
  - `restoreDefault(request, token, ruleTypeId)`: publishes the `default` from `GET api/rules/types`.

- [ ] **Step 1: Write the failing tests**
  - Unit `OrderRecordDeserializesDroppedItems`
  - `PromotionTest.spec.ts` "published promotion is shown and charged": open "Adventurer GPS Watch" and read its item id from the URL. Publish PRM-001 with a 10 % line promotion on that item, labelled "E2E 10% off". `expect.poll` (timeout 70 s, reloading) until the item page shows `$179.99`. Add to cart; the cart shows "E2E 10% off" and `$179.99`. Check out; `/user/orders` shows the new order at `$179.99`. `afterAll`: `restoreDefault("PRM-001")`.
  - `PartialFulfilmentTest.spec.ts` "dropped item is shown on orders": publish ORD-013 `{"policy":"PartialFulfilment"}`, then wait 70 s for propagation. Add "Summit Pro Harness" and "Adventurer GPS Watch". Set the watch quantity to 101 (more than the seeded `AvailableStock` of 100) and check out. Poll `/user/orders` (timeout 4 min, reloading) until "Removed (out of stock): Adventurer GPS Watch" shows. `afterAll`: `restoreDefault("ORD-013")`. If the cart caps quantity below 101, use the cart's maximum and say so in the PR. This test covers the WebApp half of the spec's partial-fulfilment AC.
- [ ] **Step 2: Run** `dotnet test --project tests/Application.UnitTests --filter "FullyQualifiedName~WebAppPricingTests"`, then (with `aspire run` up) `npx playwright test e2e/PromotionTest.spec.ts e2e/PartialFulfilmentTest.spec.ts`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** the same commands, then the whole `npm run test:e2e`. Expected: PASS, and `BrowseItemTest`, `CartTest`, and `CheckoutTest` stay green (J1–J3).
- [ ] **Step 5: Commit** `feat(webapp): show promotions and items removed by partial fulfilment`

### Task 11: ClientApp (MAUI) — unfulfillable lines on order detail

**Files:**
- Modify: `src/ClientApp/Models/Orders/OrderItem.cs` (`[JsonPropertyName("isUnfulfillable")] public bool IsUnfulfillable`; `Total => IsUnfulfillable ? 0 : Quantity * UnitPrice - Discount`)
- Modify: `src/ClientApp/Views/OrderDetailView.xaml` (the item template, ≈ line 174, shows a `Label` with the text "Out of stock – removed", visible when `IsUnfulfillable`)
- Test: `tests/ClientApp.UnitTests/Models/OrderItemTests.cs` (new)

**Interfaces:**
- Consumes: `Orderitem.IsUnfulfillable` from `GET api/orders/{id}` (Task 8). `Order.Total` already binds the server total, which is reduced after partial fulfilment.

- [ ] **Step 1: Write failing tests:** `OrderItemDeserializesUnfulfillableFlag` (JSON `{"isUnfulfillable":true}`), `UnfulfillableItemTotalIsZero`, `ItemTotalSubtractsDiscount`.
- [ ] **Step 2: Run** `dotnet test --project tests/ClientApp.UnitTests`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit** `feat(clientapp): show items removed by partial fulfilment`

### Task 12: Architecture docs (eShopOrgMeta)

**Files (eShopOrgMeta):**
- `architecture/business-rules-catalog.md`:
  - ORD-001 mechanism → "Runtime rule `ORD-001` (BusinessRules.API), resolved per order into `Order.GracePeriodEndsAt`; default 1 min"
  - ORD-013 → "Runtime rule `ORD-013`, stored per order; `AllOrNothing` | `PartialFulfilment`"
  - PRC-001, PRM-001, SHP-001 status → Adopted, with the P2-D2/P2-D3 selection rules
  - Remove the line-73 note; update gaps G-01 and G-03 to "phase 2 done"
- `architecture/domain-glossary.md`: promotion, shipping fee, stock shortfall policy, partial fulfilment, unfulfillable order item → Adopted, with owners
- `architecture/integration-events.md`: `OrderStatusChangedToPaidIntegrationEvent` notes "`OrderStockItems` excludes unfulfillable lines"
- `architecture/environments-and-deployment.md`: remove `BackgroundTaskOptions:GracePeriodTime`
- `standards/data-retention.md`: new order columns (Task 6) under the order tiers
- `specs/cross-repo/2026-11-03-runtime-business-rules-and-experiments-design.md` front matter: add the phase 2 PR to `implemented_in` once merged

- [ ] **Step 1:** Make the edits. Cite eShop file paths as sources.
- [ ] **Step 2:** Run `npx markdownlint-cli2 "architecture/**/*.md" "standards/**/*.md"` if the repo has a lint config; otherwise skip. Expected: no new errors.
- [ ] **Step 3: Commit** `docs(architecture): record production rule types and partial fulfilment`

### Phase 2 exit checks

- [ ] `dotnet build eShop.Web.slnf` with 0 warnings; `dotnet test --solution eShop.Web.slnf` green; `npm run test:e2e` green (J1–J3, J5 and the two new journeys).
- [ ] Spectral passes for the Ordering.API and BusinessRules.API OpenAPI documents.
- [ ] Manual check: with `aspire run`, publish PRC-001 10 % off one catalog type through Scalar. Within 60 s, catalog, item, cart, and checkout show the reduced price, and the order charges it (spec AC *T-shirt*). Then publish the default again.
- [ ] `superpowers:requesting-code-review` → `finishing-a-development-branch`. The PR links the spec, the phase 1 plan, and this plan.

## Self-review notes

- **Spec coverage (phase 2 rollout item → task):**
  - Rule types, validation, ranges → T1–T3; targeted most-specific-wins → T2, T4, T5
  - Calculator steps, best single promotion per level, allocation, ORD-010 → T4–T5; rounding → T4
  - ORD-001 per order → T6 (domain), T7 (resolve), T9 (OrderProcessor)
  - ORD-013 incl. partial fulfilment → T6–T8; paid event excludes unfulfillable lines → T6/T8
  - WebApp/ClientApp display → T10–T11; docs → T12
- **Acceptance criteria covered here:**
  - T-shirt 10 % → T7 (API) + T10 (UI) + exit manual check
  - Future `EffectiveFrom` → T4
  - Partial fulfilment one of three → T6 + T8 + T10
  - ORD-001 = 10 min → T6 + T9
  - J1–J3/J5 green → exit checks
  - Covered by phase 1 and not repeated: audit of who published, tampered price, price-changed prompt
- **Out of scope (later phases):** experiments and overrides (phase 4), eShopAdmin UI (phase 3), tracking (phase 5).
- **Type names are consistent across tasks:** `RuleType<T>`, `RuleTarget`/`RuleTargetKind`, `GracePeriodRuleType`, `StockShortfallPolicyRuleType`/`ShortfallPolicy`, `ShippingFeeRuleType`, `PriceAdjustmentRuleType`, `PromotionRuleType`, `DiscountAllocation.Allocate`, `PricedLine.OrderDiscountShare`, `PriceQuote.OrderPromotionLabel`, `Order.SetFulfilmentRules`, `Order.ApplyStockRejection`, `StockShortfallPolicy`, `IOrderFulfilmentRulesResolver`, `OrderSummary.DroppedItems`, `TestRulesSnapshotSource`, `PublishSnapshotAsync`.
