# 04 — API Layer

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This document specifies the HTTP surface: routes, the normative middleware order, the response envelope, pagination, filtering, relation expansion, and the upload flow. Every handler composes the interfaces of `02-core-interfaces.md`; no handler builds SQL or bypasses `Document.Set`.

## Route Map

| Prefix | Principal | Content |
|---|---|---|
| `/api/admin/auth/*` | none → session | `login`, `logout`, `csrf` (session bootstrap per BR-AUTH-1). |
| `/api/admin/users`, `/api/admin/api-keys` | session | Admin user and key management (P-1/P-2 per persona limits). |
| `/api/admin/collections` | session | Schema management — the `schema.Engine` operations of `03-dynamic-schema.md`. |
| `/api/admin/collections/{slug}/records` | session | Full CRUD, trash view, restore, purge, revisions, publish/unpublish. |
| `/api/admin/media` | session | `presign`, `finalize`, listing. |
| `/api/v1/auth/*` | none → JWT | End-user `register`, `login`, `refresh`, `logout`, `password-reset/request`, `password-reset/confirm` (F-11, BR-AUTH-13). |
| `/api/v1/collections/{slug}/records` | API key or JWT or anonymous | Published-scope reads; writes per API-key scope or end-user access rules. |
| `/recover` | none | Active only while recovery mode is enabled (BR-AUTH-12); 404 otherwise. Consumes the single-use super-admin recovery token, resets the target user's password, and revokes their sessions. |
| `/healthz` | none | Liveness (`09-deployment.md`). |
| `/readyz` | none | Readiness — database ping; see `08-observability.md`, `09-deployment.md`. |
| `/*` | none | Embedded SPA — hashed assets immutable, `index.html` no-cache. |

The envelope and error registry are v1-stable and evolve additively only.

## Middleware Order (Normative)

```text
RequestID → Logger → Recover → RateLimit → Auth
  admin routes:   → RequireSession → RequireCSRF (mutations) → RequireRecentAuth (destructive)
  public routes:  → (Principal already resolved: api_key | end_user | anonymous)
→ handler
```

The order is a business-rule surface: rate limiting precedes authentication so credential stuffing burns the limiter, not Argon2id time (BR-AUTH-6); CSRF checks run only after a valid session exists (BR-AUTH-4); recent-auth gates bind to the destructive route set (BR-AUTH-5, BR-SCHEMA-7). Changing this order requires a BUSINESS_RULES review.

## Response Envelope

Success (pagination `total` shown because the request carried `?count=exact`):

```json
{ "data": ..., "meta": { "pagination": { "limit": 25, "offset": 0, "total": 1042 } } }
```

`meta.pagination.total` appears only when the request includes `?count=exact`; public consumers default to no count (an unqualified `COUNT` is an expensive scan), and the admin UI requests it where its tables need totals.

Error — one shape everywhere, written only by `httpapi.WriteError` (BR-API-3):

```json
{ "error": { "code": "conflict", "message": "version mismatch", "details": [ { "field": "version", "expected": 7 } ] } }
```

| Code | HTTP | Producers |
|---|---|---|
| `validation_failed` | 422 | `Document.Set` field errors; safe-conversion rejections (EC-3). |
| `unauthorized` | 401 | Missing/invalid credentials; revoked refresh family (BR-AUTH-9). |
| `forbidden` | 403 | `Decision.Allowed == false`; CSRF failure; contributor publish attempts. |
| `not_found` | 404 | Unknown slug, record, or revision; stale path after rename (EC-4). |
| `conflict` | 409 | Optimistic-lock mismatch (BR-LIFE-7); trash-restore collision (BR-LIFE-5); slug collisions. |
| `rate_limited` | 429 | `middleware.RateLimit` (BR-AUTH-6). |
| `payload_too_large` | 413 | Presign size over cap; request bodies over limit. |
| `internal` | 500 | Recovered panics; post-schema-change stale-plan window (EC-1, `03-dynamic-schema.md`). |

Request-body caps producing `payload_too_large`: **5 MiB** on record-write routes (`.../records` create/update, sized to accommodate the 1 MiB per-field cap with headroom), **64 KiB** on every other route (auth, presign, finalize, and all remaining non-content routes).

## Pagination (BR-API-1)

**V1 — capped offset.** `?limit=` defaults 25, clamps to 100. `?offset=` beyond 10,000 returns `422 validation_failed` — offset scans are O(offset), and the cap converts an abuse vector into a documented boundary. Every list appends the `id` tiebreaker after the requested sort, making pagination stable under concurrent writes. *(Resolves EC-11.)*

**V1 — keyset cursors (admin lists).** Admin list endpoints also accept `?cursor=`, an opaque base64 encoding of the sort key + `id` of the last row, and return `meta.pagination.next_cursor` for the next page. `cursor` and `offset` are mutually exclusive — supplying both returns `422 validation_failed`. Cursor pagination lets the admin UI page past the 10,000-row offset ceiling at O(1) per page while preserving the mandatory `id` tiebreaker and the limit clamp (`02-core-interfaces.md` invariant 7).

**V2 — public exposure of the V1 mechanism (F-27).** The public API (`/api/v1/collections/{slug}/records`) gains the same `?cursor=` / `meta.pagination.next_cursor` mechanism already built for admin lists, alongside capped offset retained for compatibility.

## Filtering and Sorting (BR-API-4)

`?filter[<field>][<op>]=<value>` with `op ∈ eq, neq, lt, lte, gt, gte, in, contains`; `?sort=<field>` / `?sort=-<field>`. Fields must exist in the schema snapshot and survive `Decision.FieldRules` visibility; violations return `422` naming the field. `ScopePublic` queries additionally accept `filter`/`sort` only on fields marked `indexed` or `unique`; a request naming any other field returns `422 validation_failed` naming the offending field. `ScopeAdmin` and `ScopeTrash` accept any schema field, still subject to `Decision.FieldRules` visibility and the query's `statement_timeout`. All composition happens inside `query.Builder` — operators are a closed set, and `contains` maps to `ILIKE` with escaped wildcards on `text` fields only.

## Caching (BR-API-5)

Anonymous `ScopePublic` GETs carry `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` and a strong `ETag`. Any request bearing `Authorization` or a cookie receives `Cache-Control: no-store` — without exception; a cacheable credentialed response is a security bug, not a performance optimization. Both cases add `Vary: Authorization, Cookie` so shared caches never conflate anonymous and credentialed results. V1 has no purge-on-publish signal, so a published change may take up to 60 seconds to reach an edge cache — an accepted propagation window, documented here rather than engineered away. V2 adds purge-on-publish webhooks, at which point cache TTLs can lengthen.

## Idempotent Creates

Public and API-key `POST` create endpoints under `/api/v1/collections/{slug}/records` accept an optional `Idempotency-Key` request header (≤128 bytes).

Supplying the same `Idempotency-Key` from the same principal again within a 24-hour window returns the original creation result instead of creating a duplicate record. Keys are tracked in `cms_idempotency_keys` (`key_hash`, `principal_id`, `record_id`, `created_at`; unique on `(key_hash, principal_id)`; see `07-data-model.md`) and purged by `jobs.Retention` 24 hours after creation.

## Relation Expansion

`?expand=<relationField>` resolves one level deep (no nesting in V1). Expanded targets load through `ScopePublic` on public routes: a target that is trashed, draft, or hidden by access rules serializes as `null` rather than leaking or erroring — the read-side half of publish-referencing-trashed semantics (BR-LIFE-6). Unexpanded relation fields serialize as the bare UUID. Expansion resolves each relation field with a single batched IN query per field, never per-row lookups. *(Resolves EC-7.)*

## Upload Flow (BR-MEDIA-1/2/3)

```text
1. POST /api/admin/media/presign  { filename, mime, size }
   → 201 { mediaId, uploadUrl, expiresAt }        size > 50 MB (V1 constant) → 413
2. Client PUTs bytes directly to storage           binary never sees them
3. POST /api/admin/media/finalize { mediaId }
   → 200 { media }                                 storage HEAD must confirm object ≤ declared size
```

Presign validates the declared `mime` against a closed, compile-time V1 allowlist — raster images, video, audio, and PDF; any other type is rejected before a presigned URL is issued (no admin-configurable allowlist exists in V1). The declared `Content-Type` is signed into the presign policy and re-verified against the storage `HEAD` response at finalize, so a client cannot swap in an undeclared type after the presign step.

Presigned URLs expire in 15 minutes and embed a `content-length-range` condition. A client that uploads but never finalizes — or presigns and never uploads — leaves a `pending` row: the hourly orphan sweep deletes the storage object (when present) and the row after 24 hours. `Document.Set` rejects `media` field values referencing non-finalized records, so no published content ever points at an unverified object. *(Resolves EC-9.)*

## Rich Text over the Wire

`richText` fields accept and return Tiptap JSONB exclusively in V1; consumers render client-side with Tiptap's libraries. V2 adds `?format=html` (F-19) rendering server-side from the same canonical JSONB — the stored representation never changes.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-7 | Expansion serializes non-published/trashed targets as `null` under `ScopePublic` (Relation Expansion) |
| EC-9 | 15-min presign expiry + finalize verification + 24 h orphan sweep (Upload Flow) |
| EC-11 | Limit clamp, offset ceiling with 422, mandatory `id` tiebreaker; V1 keyset cursors for admin lists, V2 exposes the same mechanism publicly (Pagination) |
