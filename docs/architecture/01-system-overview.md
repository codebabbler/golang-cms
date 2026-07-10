# 01 — System Overview

**Version:** 1.0 · **Last Updated:** 2026-07-07 · **Owner:** Miraj Aryal

## What the System Is

golang-cms is a headless, config-driven CMS compiled into one Go binary that serves one tenant. Administrators define collections at runtime; the schema engine provisions one real PostgreSQL table per collection. Content leaves the system through a REST API consumed by server-to-server integrations (API keys) and end-user client applications (JWTs). The admin UI is a Svelte 5 SPA embedded into the binary via `go:embed`. PostgreSQL 16+ and S3-compatible object storage are the only runtime dependencies (BR-RUNTIME-2).

## Architecture at a Glance

```text
                        ┌─────────────────────────────────────────────┐
                        │                 Go binary                   │
 Admin browser ───────► │ httpapi (chi)                               │
 API consumer ────────► │  ├─ middleware: RequestID → Logger →        │
 End-user client ─────► │  │   RateLimit → Auth (Session|APIKey|JWT)  │
                        │  │   → RequireCSRF → RequireRecentAuth      │
                        │  ├─ access.Evaluator ─ Decision{Allowed,    │
                        │  │                      Predicate}          │
                        │  ├─ services                                │
                        │  │   ├─ schema.Engine     (runtime DDL)     │
                        │  │   ├─ lifecycle.Service (rev/publish/     │
                        │  │   │                     trash)           │
                        │  │   ├─ content.Document  (Set = only       │
                        │  │   │                     write path)      │
                        │  │   ├─ media.Service     (presign/         │
                        │  │   │                     finalize)        │
                        │  │   └─ auth.*            (sessions, keys,  │
                        │  │                         JWT)             │
                        │  ├─ query.Builder (quoting, trash filter,   │
                        │  │                 predicates)              │
                        │  ├─ jobs (Retention, Publisher tickers)     │
                        │  ├─ audit.Recorder (slog sink V1,           │
                        │  │                  cms_audit_log V2)       │
                        │  └─ embedded SPA (go:embed, hashed assets)  │
                        └───────┬─────────────────────────┬───────────┘
                                ▼                         ▼
                        PostgreSQL 16+             S3-compatible storage
                        cms_* system tables        (Cloudflare R2)
                        c_<slug> collection        direct presigned PUT
                        tables + cms_revisions     uploads from clients
```

## Request Flows

**Admin mutation.** The request passes RequestID → Logger → RateLimit → session lookup (hashed token, BR-AUTH-2) → CSRF validation (BR-AUTH-4) → recent-auth check when destructive (BR-AUTH-5) → `access.Evaluator.Decide` (BR-RBAC-2) → handler → `content.Document.Set` (BR-RBAC-5) → `lifecycle.Service.Save`, which writes the live row and its revision in one transaction (BR-LIFE-1) → `audit.Recorder.Emit` (BR-AUDIT-1).

**Public / API-key read.** Auth resolves the principal (API key hash or JWT verification) → `access.Evaluator.Decide` → `query.Builder` composes the query with identifier quoting (BR-SCHEMA-3), the trash filter (BR-LIFE-4), the published-only scope (BR-API-2), and the role predicate (BR-RBAC-6) → pagination clamps apply (BR-API-1).

**Media upload.** The client requests a presigned PUT URL (size-capped, ≤15 min expiry), uploads directly to storage — bytes never transit the binary (BR-MEDIA-1) — then finalizes; only finalized records attach to `media` fields (BR-MEDIA-3).

## Data Plane

Each collection is a real table `c_<slug>` carrying seven system columns (`id`, `status`, `version`, `created_at`, `updated_at`, `created_by`, `deleted_at`). The live row holds the current published content; full history — including drafts newer than the published version — lives in `cms_revisions` as JSONB snapshots (BR-LIFE-2). Schema metadata (`cms_collections`, `cms_fields`) is cached in memory and reloads before the schema-change advisory lock releases (BR-RUNTIME-7), which is safe because exactly one process exists.

## Lifecycle Summary

Startup: embedded migrations under advisory lock → schema cache load → missed-schedule catch-up → HTTP listener (BR-RUNTIME-3). Shutdown: drain in-flight requests within 15 seconds (BR-RUNTIME-6). Background work is two in-process tickers — `jobs.Retention` (trash purge, revision pruning, orphan sweep) and `jobs.Publisher` (V2 scheduled publishing) — never external queues (BR-RUNTIME-5).

## Edge-Case Register

The register below enumerates the failure modes and boundary conditions the architecture must handle explicitly. **Citation convention:** every architecture document states which register items it resolves, inline at the resolving rule or section, using the form *(Resolves EC-n.)* A document that touches a register item's subsystem without citing it is incomplete. `BUSINESS_RULES.md` carries the authoritative resolution for EC-1, EC-2, EC-3, EC-6, EC-7, EC-8, EC-9, EC-11, and EC-13; the remaining items resolve in the documents assigned below.

| ID | Case | Resolving document |
|---|---|---|
| EC-1 | Concurrent schema change vs. in-flight writes | `03-dynamic-schema.md` (BR-SCHEMA-6) |
| EC-2 | Field drop with existing data | `03-dynamic-schema.md` (BR-SCHEMA-7) |
| EC-3 | Type-change requests outside the safe-conversion matrix | `03-dynamic-schema.md` (BR-SCHEMA-5) |
| EC-4 | Collection rename | `03-dynamic-schema.md` |
| EC-5 | Revision restore after schema drift | `03-dynamic-schema.md`, `07-data-model.md` |
| EC-6 | Trash-restore unique-constraint collision | `07-data-model.md` (BR-LIFE-5) |
| EC-7 | Publish referencing trashed targets | `04-api-layer.md` (BR-LIFE-6) |
| EC-8 | Refresh-token replay | `05-auth-security.md` (BR-AUTH-9) |
| EC-9 | Presigned upload expiry and orphaned objects | `04-api-layer.md` (BR-MEDIA-2) |
| EC-10 | Rate limiting behind a proxy (`X-Forwarded-For` trust) | `05-auth-security.md` |
| EC-11 | Deep pagination abuse | `04-api-layer.md` (BR-API-1) |
| EC-12 | FTS reindex on schema change | `03-dynamic-schema.md` |
| EC-13 | Missed scheduled publishes | `08-observability.md`, `09-deployment.md` (BR-LIFE-9) |
| EC-14 | Graceful-shutdown draining | `09-deployment.md` (BR-RUNTIME-6) |
| EC-15 | SPA cache busting | `09-deployment.md` |

## Consumer Classes

| Principal | Mechanism | Surface |
|---|---|---|
| Admin user (P-1…P-5) | Session cookie + CSRF | Admin UI, `/api/admin/*` |
| API Consumer (P-6) | `Authorization: Bearer cms_...` | `/api/collections/*` per key scope |
| End User (P-7) | RS256 JWT + refresh rotation | Public content, V3 commerce/paywalls |
