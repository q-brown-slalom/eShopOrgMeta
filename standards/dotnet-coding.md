---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop repo (global.json, Directory.Build.props, Directory.Packages.props, .editorconfig)
---

# .NET coding standard

## Toolchain

| Item | Standard | Source |
|------|----------|--------|
| SDK | .NET 10 (`10.0.302`, `rollForward: latestFeature`) pinned via `global.json` | `eShop/global.json` |
| Target framework | `net10.0` | project files |
| Orchestration | .NET Aspire 13.x (AppHost + ServiceDefaults) | `eShop/aspire.config.json` |
| Packages | Central Package Management, transitive pinning on | `eShop/Directory.Packages.props` |
| Build | `TreatWarningsAsErrors=true`, `ImplicitUsings=enable`, artifacts output | `eShop/Directory.Build.props` |
| Feeds | nuget.org only | `eShop/nuget.config` |

- Add package versions **only** in `Directory.Packages.props`; never `Version=` in a `.csproj`.
- Enable `<Nullable>enable</Nullable>` in new projects.
- MediatR stays on 13.x (pre-license-change) unless an ADR says otherwise.

## Style (from `.editorconfig`)

- 4-space indent for C#; 2-space for XML/props/csproj; UTF-8; final newline.
- Allman braces (`csharp_new_line_before_open_brace = all`); always use braces.
- `var` everywhere; predefined type names (`int`, not `Int32`); no `this.` qualification.
- Expression bodies for properties/accessors; block bodies for methods/constructors.
- Prefer `is null`, `??`, throw expressions, pattern matching.
- `System.*` usings first; shared usings in `GlobalUsings.cs`.
- Constants `PascalCase`.

## Project conventions

- Every service calls `builder.AddServiceDefaults()` (or `AddBasicServiceDefaults()` for gRPC-only)
  and `app.MapDefaultEndpoints()`.
- Service registration lives in `Extensions/Extensions.cs` (`AddApplicationServices`).
- Minimal API endpoints live in `Apis/<Name>Api.cs` as `Map<Name>Api()` extension methods.
- New services must be added to `eShop.AppHost` with explicit `.WithReference(...)` / `.WaitFor(...)`.
