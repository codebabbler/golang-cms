# golang-cms — Business Rules

**Version:** 1.0 · **Last Updated:** 2026-07-07 · **Owner:** Miraj Aryal

This manual defines the non-negotiable invariants of golang-cms. Every rule states the invariant first, then the exact enforcement point in code. Implementation, review, and tests must trace to these identifiers (see Rule-to-Code Traceability). Rules tagged **(V2)** or **(V3)** bind from that version onward; untagged rules bind from V1. Rules tagged **[structural]** hold by construction (the enforcing code path is the only path that exists) and are exempt from the test-name trace.

## 1. Tenancy & Runtime

- **BR-RUNTIME-1.** The binary serves exactly one tenant. No route pattern, header, or query parameter carries a tenant identifier. **[structural]**
  *Enforcement:* `httpapi` route registration — no tenant segment exists in any route.
- **BR-RUNTIME-2.** PostgreSQL 16+ and S3-compatible object storage are the only runtime dependencies. The binary never connects to Redis, message queues, or external caches. **[structural]**
  *Enforcement:* `main.go` dependency wiring; CI dependency review rejects new service clients.
- **BR-RUNTIME-3.** Startup executes in strict order: embedded migrations under advisory lock → schema cache load → HTTP listener.
  *Enforcement:* `app.Run` — the listener starts only after `schema.Cache.Load` returns.
- **BR-RUNTIME-4.** Rate-limiting state lives in process memory and resets on restart.
  *Enforcement:* `middleware.RateLimit` — in-memory token buckets keyed by email and client IP.
- **BR-RUNTIME-5.** Background work (retention, scheduled publishing) runs as in-process goroutine tickers.
  *Enforcement:* `jobs.Scheduler` — the only background-execution entry point.
- **BR-RUNTIME-6.** Shutdown drains in-flight requests within a 15-second window before exit; the drain drops zero accepted requests.
  *Enforcement:* `app.Run` — `http.Server.Shutdown` with a 15-second context.
- **BR-RUNTIME-7.** The in-memory schema cache reloads before the schema-change advisory lock releases; a request served after a schema change never sees stale metadata.
  *Enforcement:* `schema.Engine.Apply` — cache reload precedes lock release.

## 2. Schema Engine

- **BR-SCHEMA-1.** Every user-defined collection maps to one real Postgres table named `c_<slug>`; system tables carry the `cms_` prefix. User collections cannot collide with system tables.
  *Enforcement:* `schema.Engine.CreateCollection` — table-name assembly is code-owned; slugs never carry a prefix.
- **BR-SCHEMA-2.** Collection and field slugs match `^[a-z][a-z0-9_]{0,54}$` and do not appear in the reserved-word blocklist (which includes all system column names).
  *Enforcement:* `httpapi/admin.validateSlug` at the API boundary; `schema.Engine` re-validates before DDL (defense in depth).
- **BR-SCHEMA-3.** User input never reaches SQL as an identifier without quoting. Identifiers are never SQL parameters; values are never interpolated.
  *Enforcement:* `query.Builder.QuoteIdent` — the only identifier-interpolation path in the codebase.
- **BR-SCHEMA-4.** The schema engine executes only whitelisted DDL: CREATE TABLE, ADD COLUMN, DROP COLUMN, RENAME COLUMN, RENAME TABLE, CREATE INDEX, DROP INDEX, ADD FOREIGN KEY, DROP FOREIGN KEY, and safe type changes.
  *Enforcement:* `schema.Engine.Apply` — a closed switch over the operation set; unknown operations return an error.
- **BR-SCHEMA-5.** Field type changes succeed only inside the safe-conversion matrix: `number` precision widening and any-type→`text`. The engine rejects all other conversions with a remediation message (drop and recreate). *(Resolves EC-3.)*
  *Enforcement:* `schema.Engine.planTypeChange`.
- **BR-SCHEMA-6.** Every schema change runs inside one transaction holding `pg_advisory_xact_lock`; the DDL statement and the `cms_collections`/`cms_fields` metadata update commit atomically. *(Resolves EC-1.)*
  *Enforcement:* `schema.Engine.Apply` — single-transaction execution.
- **BR-SCHEMA-7.** Destructive schema changes (DROP COLUMN, dropping a collection) require re-authentication within the 4-hour window plus typed confirmation of the target slug. Dropping a field removes live values while `cms_revisions` snapshots retain the dropped data. *(Resolves EC-2.)*
  *Enforcement:* `middleware.RequireRecentAuth` + `httpapi/admin` confirmation validator; retention via JSONB snapshots in `lifecycle.Service.Save`.
- **BR-SCHEMA-8.** System column names (`id`, `status`, `version`, `created_at`, `updated_at`, `created_by`, `deleted_at`) are reserved and rejected as field slugs.
  *Enforcement:* `httpapi/admin.validateSlug` blocklist.

## 3. Content Lifecycle

- **BR-LIFE-1.** Every record write appends a revision with a monotonic `version_no` in the same transaction as the live-row write. History is append-only; no code path updates or deletes a revision row except retention pruning.
  *Enforcement:* `lifecycle.Service.Save` — revision insert and live-row write share one transaction.
- **BR-LIFE-2.** The live row holds the current published content for published records; drafts newer than the published version exist only in `cms_revisions`. Publishing copies the revision's data into the live row.
  *Enforcement:* `lifecycle.Service.Publish`.
- **BR-LIFE-3.** Publishing and unpublishing require role `editor` or above.
  *Enforcement:* `access.Evaluator.Decide` — `publish` action check in the lifecycle handler.
- **BR-LIFE-4.** Every collection query filters `deleted_at IS NULL` unless the caller explicitly requests the trash view (admin-only).
  *Enforcement:* `query.Builder` — the trash predicate is part of the base query and cannot be omitted by callers.
- **BR-LIFE-5.** Purge of a trashed record fails while foreign keys reference it (FK RESTRICT). Restoring a trashed record whose unique field now collides with a live record returns `409 Conflict` naming the conflicting field; restore never overwrites. *(Resolves EC-6.)*
  *Enforcement:* Postgres FK RESTRICT constraints + `lifecycle.Service.Restore` unique-violation mapping.
- **BR-LIFE-6.** Publishing a record whose relation targets a draft or trashed record succeeds; the public API resolves such references to `null` and omits the targets. *(Resolves EC-7.)*
  *Enforcement:* `query.Builder` relation join predicate — `status = 'published' AND deleted_at IS NULL`.
- **BR-LIFE-7.** Record updates require the current `version` value; a mismatch returns `409 Conflict` and writes nothing.
  *Enforcement:* `lifecycle.Service.Save` — compare-and-set `WHERE version = $expected`.
- **BR-LIFE-8.** The retention job purges trash older than `CMS_TRASH_RETENTION_DAYS` and prunes revisions beyond `CMS_REVISION_LIMIT` per record, oldest first. Pruning never removes the currently published revision.
  *Enforcement:* `jobs.Retention` — pruning query excludes the published `version_no`.
- **BR-LIFE-9 (V2).** Scheduled publishing fires within 60 seconds of `publish_at`; on startup the system publishes every record whose `publish_at` elapsed during downtime. *(Resolves EC-13.)*
  *Enforcement:* `jobs.Publisher` ticker + startup catch-up scan in `app.Run`.

## 4. Authentication

- **BR-AUTH-1.** Admin login sets `cms_session=<256-bit-random>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800` and returns `csrfToken` in the body.
  *Enforcement:* `auth.SessionService.Issue`.
- **BR-AUTH-2.** The database stores only hashed session tokens (`token_hash, user_id, created_at, last_seen_at, ip, user_agent`); the raw cookie value never persists.
  *Enforcement:* `auth.SessionService` — hash before insert, hash before lookup.
- **BR-AUTH-3.** Password hashing uses Argon2id with per-hash salts.
  *Enforcement:* `auth.Password.Hash` / `auth.Password.Verify`.
- **BR-AUTH-4.** Every state-changing admin request carries `X-CSRF-Token` validated against the session record; requests without it fail with `403`.
  *Enforcement:* `middleware.RequireCSRF` on all admin mutation routes.
- **BR-AUTH-5.** Sessions expire after 7 idle days and 30 absolute days. Destructive operations require re-authentication within the preceding 4 hours.
  *Enforcement:* `middleware.RequireSession` (expiry) + `middleware.RequireRecentAuth` (destructive routes).
- **BR-AUTH-6.** Login attempts rate-limit at 10/15 min per email and 30/15 min per IP.
  *Enforcement:* `middleware.RateLimit` on the login route.
- **BR-AUTH-7.** API keys display once at creation; the database stores `sha256(token)`. Keys authenticate as `Authorization: Bearer cms_...`, carry per-collection scopes, and revocation sets `revoked_at` while the row remains for audit.
  *Enforcement:* `auth.APIKeyService` + scope check in the collections handler.
- **BR-AUTH-8.** Public JWTs are RS256 with 15-minute TTL and carry identity only; the server resolves permissions on every request. RS256 stands for client-library compatibility; the system revisits EdDSA only on a concrete constraint.
  *Enforcement:* `auth.JWTService.Issue` (claims shape) + `access.Evaluator.Decide` per request.
- **BR-AUTH-9.** Refreshing rotates both tokens. Presenting an already-rotated refresh token revokes the entire token family and returns `401`. *(Resolves EC-8.)*
  *Enforcement:* `auth.JWTService.Refresh` — family check precedes issuance.
- **BR-AUTH-10.** The RSA-2048 keypair persists in `cms_system_keys` (the system-table form of the auth specification's `system_keys`); the system auto-generates it when `JWT_PRIVATE_KEY` is absent.
  *Enforcement:* `auth.Keys.Load` at startup.
- **BR-AUTH-11.** When `cms_users` is empty at startup, the system logs a single-use setup token and enables `/setup`; consuming it creates the first super admin and disables the route. The token dies on use or process exit.
  *Enforcement:* `auth.Bootstrap` at startup + the `/setup` handler.

## 5. Access Control

- **BR-RBAC-1.** Exactly five roles exist: `super_admin`, `admin`, `editor`, `contributor`, `viewer`.
  *Enforcement:* CHECK constraint on `cms_users.role` + `access` package constants.
- **BR-RBAC-2.** Per-collection access rules live as JSONB config for `read`, `create`, `update`, `delete`; the evaluator returns `Decision{Allowed, Predicate}` and every collection handler calls it before building a query.
  *Enforcement:* `access.Evaluator.Decide` — handlers receive the query builder only through `query.Builder.WithDecision`.
- **BR-RBAC-3.** A missing access rule denies. No default-allow path exists.
  *Enforcement:* `access.Evaluator.Decide` default branch.
- **BR-RBAC-4.** Field-level rules `hideFromRoles` and `readOnlyForRoles` apply on both read serialization and write validation.
  *Enforcement:* `content.Document.Set` (writes) + the response serializer (reads).
- **BR-RBAC-5.** Mass assignment is impossible: every write passes through `content.Document.Set`, which drops unknown fields and rejects role-read-only fields against the cached schema.
  *Enforcement:* `content.Document.Set` — the only write path into collection tables.
- **BR-RBAC-6.** Row-scope predicates (e.g., `ownerOnly`) compile into every list and read query for the deciding role.
  *Enforcement:* `query.Builder.WithDecision` — appends the `Decision.Predicate`.

## 6. Media

- **BR-MEDIA-1.** Uploads go directly to object storage via presigned PUT URLs; the binary never proxies upload bytes. **[structural]**
  *Enforcement:* `media.Service.Presign` — no route accepts an upload body.
- **BR-MEDIA-2.** Presigned URLs expire within 15 minutes and carry a content-length-range cap. The orphan sweep deletes storage objects with no finalized media record after 24 hours. *(Resolves EC-9.)*
  *Enforcement:* `media.Service.Presign` policy + `jobs.Retention` orphan sweep.
- **BR-MEDIA-3.** Media records finalize only after the client confirms upload; a media field value always references a finalized record.
  *Enforcement:* `media.Service.Finalize` + `content.Document.Set` reference validation.
- **BR-MEDIA-4.** The binary never processes pixels; transformation is Cloudflare Image Resizing's job. **[structural]**
  *Enforcement:* dependency policy (BR-RUNTIME-2) — no imaging library exists in `go.mod`.

## 7. API Conduct

- **BR-API-1.** List endpoints clamp `limit` to 100 (default 25), always append the stable `id` tiebreaker sort, and reject `offset` greater than 10,000 with `400`. *(Resolves EC-11.)*
  *Enforcement:* `httpapi.ParsePagination` — shared by all list handlers.
- **BR-API-2.** Public and API-key reads return only published, non-trashed records unless the key scope explicitly grants draft access.
  *Enforcement:* `query.Builder` public-scope base predicate.
- **BR-API-3.** Every error response uses the shared envelope `{"error": {"code", "message", "details"}}`.
  *Enforcement:* `httpapi.WriteError` — the only error-writing function.

## 8. Audit

- **BR-AUDIT-1.** Every mutation (schema change, record write, publish, trash, purge, permission change, key lifecycle) emits an audit event with actor, action, entity, and timestamp through the audit recorder.
  *Enforcement:* `audit.Recorder.Emit` — called by `schema.Engine`, `lifecycle.Service`, `auth.APIKeyService`, and the admin user handlers.
- **BR-AUDIT-2.** The V1 sink writes audit events to `slog`; V2 adds the persistent `cms_audit_log` sink and admin UI without touching call sites.
  *Enforcement:* `audit.Recorder` sink configuration — call sites depend only on the interface.
- **BR-AUDIT-3 (V2).** Persisted audit records are append-only; no API mutates or deletes them.
  *Enforcement:* `cms_audit_log` — no UPDATE/DELETE statements exist against it; sqlc queries expose insert and select only.

## Naming Constants

| Variable | Default | Purpose |
|---|---|---|
| `DATABASE_URL` | — | PostgreSQL connection string. |
| `CMS_SESSION_SECRET` | — | Cookie signing entropy. |
| `JWT_PRIVATE_KEY` | Auto-generated | RSA-2048 PEM for RS256. |
| `JWT_PUBLIC_KEY` | Derived from private | RS256 public verification. |
| `S3_ENDPOINT` | — | S3-compatible API endpoint. |
| `S3_BUCKET` | — | Raw media storage bucket name. |
| `S3_ACCESS_KEY` | — | S3-compatible Access Key ID (R2 Token). |
| `S3_SECRET_KEY` | — | S3-compatible Secret Access Key (R2 Secret). |
| `R2_ACCOUNT_ID` | — | Cloudflare R2 Account ID (required when using R2). |
| `R2_PUBLIC_BUCKET_URL` | — | Public custom domain or dev URL for direct asset delivery. |
| `CMS_PORT` | `8080` | HTTP server bind port. |
| `CMS_LOG_LEVEL` | `info` | `slog` level (debug, info, warn, error). |
| `CMS_TRASH_RETENTION_DAYS` | `30` | Days a trashed record persists before auto-purge. |
| `CMS_REVISION_LIMIT` | `50` | Maximum revisions retained per record. |

## Edge-Case Coverage (Batch 1)

| EC | Resolved by |
|---|---|
| EC-2 | BR-SCHEMA-7 |
| EC-6 | BR-LIFE-5 |
| EC-7 | BR-LIFE-6 |
| EC-8 | BR-AUTH-9 |
| EC-11 | BR-API-1 |

(BR-SCHEMA-5, BR-SCHEMA-6, BR-LIFE-9, and BR-MEDIA-2 additionally resolve EC-3, EC-1, EC-13, and EC-9.)

## Rule-to-Code Traceability

Every non-structural rule maps to at least one Go test whose name embeds the rule identifier with underscores replacing hyphens:

- Test function form: `func TestBR_SCHEMA_7_DestructiveChangeRequiresRecentAuth(t *testing.T)`
- Subtest form: `t.Run("BR-LIFE-5 restore collision returns 409", ...)`

The CI job `trace` extracts every `BR-` identifier from this manual, greps `_test.go` files for each, and fails the build on any non-structural rule without a matching test. Rules tagged **[structural]** are exempt: their enforcement is the absence of an alternative code path, which review — not tests — verifies.
