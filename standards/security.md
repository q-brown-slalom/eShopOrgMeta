---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/Identity.API/Configuration/Config.cs, eShop/.config/CredScanSuppressions.json
---

# Security standard

## Identity and access

- **Identity.API** (Duende IdentityServer) is the only token issuer. Services validate JWTs via
  `AddDefaultAuthentication()` from ServiceDefaults.
- Each API owns a scope (`orders`, `basket`, `webhooks`, …). New APIs register a new scope + client
  in `Config.cs`; least privilege per client.
- Interactive clients use Authorization Code **with PKCE** in all non-local environments.
- Admin capabilities (e.g. the planned business-rules admin app) require a dedicated scope and
  role-based policy — never reuse customer scopes.

## Secrets

- No secrets in source, `appsettings*.json`, or test data. Local dev: user secrets / Aspire parameters.
  Deployed: managed secret store (e.g. Key Vault) referenced via Aspire.
- Demo/test credentials must be listed in `.config/CredScanSuppressions.json` with justification.

## Data protection

- Never log card numbers, CVV, tokens, or passwords. Mask PII in logs and traces.
- Card data is tokenized by the payment provider. Store only the token, brand, last 4 digits, and expiry
  ([ADR-0007](../decisions/0007-tokenize-card-data.md)).
- Retention and erasure follow [data-retention.md](data-retention.md).
- Payment data handling must be reviewed by a second CODEOWNER (see PR rules).

## OWASP Top 10 checklist for specs and reviews

- [ ] AuthZ enforced server-side on every endpoint (broken access control).
- [ ] Input validated (FluentValidation / model binding); parameterized queries only (EF/Dapper params).
- [ ] No sensitive data in URLs, logs, or error responses (ProblemDetails without stack traces in prod).
- [ ] Dependencies current; Dependabot alerts triaged.
- [ ] SSRF: outbound URLs (e.g. webhook targets) validated against allow rules.
- [ ] Security-relevant events logged for audit.
