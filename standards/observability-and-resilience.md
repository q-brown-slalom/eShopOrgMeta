---
status: Adopted
owner: "Q Brown"
last_reviewed: 2026-11-02
source: eShop/src/eShop.ServiceDefaults/Extensions.cs
---

# Observability and resilience standard

All services get these via `AddServiceDefaults()`; don't reimplement them per service.

## Telemetry (OpenTelemetry)

- Logs, metrics, and traces exported over OTLP (`OTEL_EXPORTER_OTLP_ENDPOINT`); viewed in the
  Aspire dashboard locally.
- Structured logging with message templates — no string interpolation in log calls.
- New background flows and event handlers create an `Activity` and set tags for business IDs
  (e.g. `order.id`), never PII.
- Register custom `ActivitySource`/`Meter` names in ServiceDefaults.

## Health

- `/health` (readiness) and `/alive` (liveness) via `MapDefaultEndpoints()` — mapped in
  **Development only**. Exposing them in other environments needs an ADR (security implications).
- Every new external dependency (DB, cache, bus, HTTP API) adds a health check.

## Resilience

- Outbound `HttpClient`s use `AddStandardResilienceHandler()` (retry, timeout, circuit breaker).
- Service-to-service addresses use Aspire service discovery (`http://<resource-name>`), never
  hard-coded hosts.
