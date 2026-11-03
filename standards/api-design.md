---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/Catalog.API/Apis/CatalogApi.cs, eShop/src/Ordering.API/Apis/OrdersApi.cs, eShop/.spectral.yml
---

# API design standard

## HTTP APIs

- **Minimal APIs only** (no MVC controllers). Group routes with `MapGroup("api/<resource>")`.
- **Versioning:** `Asp.Versioning`; declare with `NewVersionedApi()` + `HasApiVersion(x, 0)`.
  Clients send `api-version` as a query string. `ReportApiVersions = true`.
  Breaking changes ship as a new version; old version stays until consumers migrate.
- **Results:** return `TypedResults.*` with explicit unions, e.g. `Results<Ok<Order>, NotFound>`.
- **Errors:** RFC 7807 `ProblemDetails` (`AddProblemDetails()`); validation errors as `400`.
- **Metadata:** every endpoint has `.WithName()`, `.WithSummary()`, `.WithDescription()`, `.WithTags()`;
  every parameter has a description.
- **Auth:** `.RequireAuthorization()` on groups that touch user data; scope per service (see
  [security.md](security.md)).
- **Idempotency:** state-changing commands accept `x-requestid` (Guid) header and use the
  `IdentifiedCommand` + `RequestManager` pattern.
- **Parameter grouping:** use `[AsParameters]` service records instead of long parameter lists.
- **Pagination:** `PaginatedItems<T>` (`PageIndex`, `PageSize`, `Count`, `Data`), `pageIndex`/`pageSize`
  query params.
- **Docs UI:** OpenAPI via `AddDefaultOpenApi()`; Scalar UI.

## OpenAPI lint (Spectral, from `.spectral.yml`)

Extends `spectral:oas` plus:

| Rule | Requirement |
|------|-------------|
| `success-response` | Each operation declares 200/201/204 |
| `error-response` | Each operation declares 400 and/or 404 |
| `parameter-description` | Every parameter has a description |

## gRPC

- Use gRPC for low-latency internal calls (Basket pattern). Protos live in `Proto/`.
- Proto changes are additive only; never renumber or reuse field numbers.

## Integration events

See [architecture-patterns.md](architecture-patterns.md#integration-events) and
[architecture/integration-events.md](../architecture/integration-events.md).
