# 08 — Observability

**Version:** 1.0 · **Last Updated:** 2026-07-08 · **Owner:** Miraj Aryal

Structured logs are the telemetry surface. The binary ships no metrics endpoint and no tracing dependency in V1 — adding either would breach the dependency invariant's spirit of operational minimalism (BR-RUNTIME-2); log-derived dashboards cover the single-tenant operational questions.

## Logging Conventions (N-8)

- Handler: `slog` JSON to stdout; level from `CMS_LOG_LEVEL` (`debug`, `info`, `warn`, `error`).
- **One line per request**, emitted by `middleware.Logger` at completion:

```json
{"level":"info","msg":"request","request_id":"...","principal_kind":"admin","principal_id":"...","method":"POST","route":"/api/admin/collections/{slug}/records","status":200,"duration_ms":12,"bytes":1834}
```

- Routes log as chi patterns, not raw paths — no record IDs or slugs leak into aggregatable fields; raw paths appear only at `debug`.
- Secrets never log: no tokens, cookie values, password material, presigned URLs, or JWT bodies at any level. The API-key `cms_` prefix makes accidental leakage greppable in CI log-assertion tests.
- Recovered panics log at `error` with stack traces and the request ID, then return the `internal` envelope (`04-api-layer.md`).

## Request Correlation

`middleware.RequestID` assigns a UUIDv7 per request, stores it in context, returns it as `X-Request-ID`, and every downstream log line and audit event carries it. A support report quoting the header value therefore joins request log, audit events, and any error stack in one grep.

## Audit Event Stream (BR-AUDIT-1, BR-AUDIT-2)

V1 sink — a distinguished `slog` line:

```json
{"level":"info","msg":"audit","request_id":"...","actor_kind":"admin","actor_id":"...","action":"schema.field.drop","entity":"collection:posts/field:summary","detail":{"confirmed_slug":"summary"},"at":"..."}
```

- `action` follows `domain.entity.verb` (`schema.collection.create`, `content.record.publish`, `auth.key.revoke`) — a closed vocabulary listed with the audit package so V2's UI filters enumerate it.
- Restore operations record drift-mapping outcomes — skipped fields, defaulted fields — in `detail` (EC-5, `03-dynamic-schema.md`).
- V2 adds the `cms_audit_log` sink behind the same interface; the `slog` line remains, so log-based alerting survives the upgrade.

## Job Telemetry

Both tickers log start/finish with counts and durations:

- `jobs.Retention` (hourly): trashed rows purged, purges skipped on FK RESTRICT (each skip names the blocking reference), revisions pruned, media orphans swept.
- `jobs.Publisher` (V2, every 30 s): records published on schedule.

**Missed-schedule catch-up** *(Resolves EC-13, observability half)*: the startup catch-up scan logs one `warn` line per late publication with `scheduled_at`, `published_at`, and the delay — downtime-induced lateness is visible and alertable, never silent. `09-deployment.md` owns the startup-ordering half.

## Schema-Change Visibility

Every `schema.Engine.Apply` logs the operation, duration, and — for V2 search rebuilds — the tsvector regeneration time (EC-12, `03-dynamic-schema.md`), giving operators the data to schedule heavy schema changes.

## Health

`/healthz` returns 200 once startup completes (migrations, schema cache, catch-up — BR-RUNTIME-3) and the database ping succeeds; it returns 503 during drain (EC-14, `09-deployment.md`). It is the only unauthenticated non-SPA endpoint.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-13 | Per-record `warn` logging of late publishes at startup catch-up (Job Telemetry) |
