---
description: "Use when editing C# in any eShop org repo: coding, API, architecture, testing standards."
applyTo: "**/*.cs"
---

Follow the org standards in eShopOrgMeta:

- `standards/dotnet-coding.md` — CPM, warnings as errors, `.editorconfig` style, ServiceDefaults.
- `standards/api-design.md` — Minimal APIs, Asp.Versioning, TypedResults, ProblemDetails, endpoint metadata.
- `standards/architecture-patterns.md` — service-owned data, DDD/CQRS where rules are rich, outbox + idempotent handlers.
- `standards/testing.md` — TDD; MSTest + NSubstitute; functional tests via WebApplicationFactory.
- `standards/security.md` — scoped JWT auth, no secrets/PII in code or logs.
