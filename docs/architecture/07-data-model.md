# 07 — Data Model

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This document specifies physical storage: naming, every system table, the anatomy of collection tables, the live-table/revisions contract, trash semantics, and retention bounds. The engine that mutates this model is specified in `03-dynamic-schema.md`.

## Naming (BR-SCHEMA-1)

| Prefix | Owner | Examples |
|---|---|---|
| `cms_` | System tables, migrated at startup via embedded migrations (BR-RUNTIME-3) | `cms_users`, `cms_revisions` |
| `c_` | One table per user-defined collection, managed by `schema.Engine` | `c_posts` |
| `cj_` | Join tables for many-to-many relations (V2) | `cj_posts_tags` |

Join-table names follow `cj_<collectionSlug>_<fieldSlug>`; when the result exceeds Postgres's 63-byte identifier limit, each slug truncates to its first 20 characters and the name gains an 8-character hash suffix of the full pair, keeping names deterministic and collision-free.

## System Tables (V1)

Types abbreviated; all `id` columns are UUIDv7 generated application-side (BR-RUNTIME context, `../BUSINESS_RULES.md`).

| Table | Columns | Notes |
|---|---|---|
| `cms_users` | `id`, `email` (unique), `password_hash`, `role`, `created_at`, `updated_at` | `role` CHECK-constrained to the five roles (BR-RBAC-1); Argon2id hashes (BR-AUTH-3). |
| `cms_sessions` | `token_hash` (PK), `user_id` FK, `csrf_hash`, `created_at`, `last_seen_at`, `ip`, `user_agent` | Raw tokens never persist (BR-AUTH-2); CSRF validates against `csrf_hash` (BR-AUTH-4). |
| `cms_api_keys` | `id`, `name`, `token_hash` (unique), `scopes` JSONB, `created_by` FK, `created_at`, `revoked_at` | Revocation keeps the row for audit (BR-AUTH-7). |
| `cms_end_users` | `id`, `email` (unique), `password_hash`, `created_at`, `updated_at` | The JWT user store (F-11); separate from `cms_users` because the threat models differ. |
| `cms_refresh_tokens` | `id`, `family_id`, `end_user_id` FK, `token_hash` (unique), `issued_at`, `rotated_at`, `revoked_at` | `family_id` groups rotations; reuse revokes the family (BR-AUTH-9). |
| `cms_system_keys` | `name` (PK), `private_pem`, `public_pem`, `created_at` | RS256 keypair, auto-generated when `JWT_PRIVATE_KEY` is absent (BR-AUTH-10). |
| `cms_collections` | `id`, `slug` (unique), `name`, `access_rules` JSONB, `search_config` JSONB (V2), `created_at`, `updated_at` | Access rules per BR-RBAC-2; all references to a collection use `id`, never `slug` (EC-4, `03-dynamic-schema.md`). |
| `cms_fields` | `id`, `collection_id` FK, `slug`, `type`, `config` JSONB, `position`, `created_at` | Unique on `(collection_id, slug)`. |
| `cms_revisions` | `id`, `collection_id` FK, `record_id`, `version_no`, `data` JSONB, `published` bool, `created_by`, `created_at` | Unique on `(collection_id, record_id, version_no)`; partial unique on `(collection_id, record_id)` WHERE `published` — at most one published revision per record. |
| `cms_media` | `id`, `object_key` (unique), `mime`, `size_bytes`, `metadata` JSONB, `status` (`pending`\|`finalized`), `created_by`, `created_at`, `finalized_at` | `pending` rows older than 24 h and their objects fall to the orphan sweep (BR-MEDIA-2). |
| `cms_reset_tokens` | `id`, `user_kind` (`admin`\|`end_user`), `user_id`, `token_hash` (unique), `expires_at`, `used_at`, `created_by`, `created_at` | Backs admin-issued resets, end-user password reset, and BR-AUTH-13; single-use, 30-minute expiry; used/expired rows purged by `jobs.Retention`. |
| `cms_idempotency_keys` | `key_hash`, `principal_id`, `record_id`, `created_at` | Unique on `(key_hash, principal_id)`; backs `Idempotency-Key` on public/API-key creates (`04-api-layer.md`); purged by `jobs.Retention` 24 h after creation. |

V2 adds `cms_audit_log` (append-only — BR-AUDIT-3), `cms_redirects`, `cms_webhooks`, and `cms_webhook_deliveries` (the webhook delivery-retry outbox — durable across restarts, Postgres-only, BR-RUNTIME-2-compatible). V3 adds `cms_carts` and `cms_orders`. Their layouts are specified in their delivery cycles; their names are reserved now.

## Collection Table Anatomy

Every `c_<slug>` table carries exactly these system columns (BR-SCHEMA-8) plus one column per user field:

| Column | Type | Constraints |
|---|---|---|
| `id` | `UUID` | PK, UUIDv7, application-generated |
| `status` | `TEXT` | CHECK `('draft','published')` |
| `version` | `BIGINT` | optimistic-lock counter, increments every write (BR-LIFE-7) |
| `created_at` | `TIMESTAMPTZ` | NOT NULL |
| `updated_at` | `TIMESTAMPTZ` | NOT NULL |
| `created_by` | `UUID` | actor reference |
| `deleted_at` | `TIMESTAMPTZ` | NULL = live; non-NULL = trashed (BR-LIFE-4) |

User-field storage follows the field-type reference: `text→TEXT`, `richText→JSONB`, `number→NUMERIC`, `boolean→BOOLEAN`, `datetime→TIMESTAMPTZ`, `media→UUID` FK to `cms_media` with `ON DELETE RESTRICT` plus the delivery URL resolved at read time, `relation→UUID` FK to the target's `id` with `ON DELETE RESTRICT`, `json→JSONB`. User columns are always nullable in DDL; `required` enforcement lives in `content.Document.Set` (`03-dynamic-schema.md`).

**Indexes.** PK on `id`; a partial index on `(status) WHERE deleted_at IS NULL` serving the public published-only scope (BR-API-2); a composite `(field, id)` B-tree per `indexed` field, serving the mandatory `id` tiebreaker sort directly (`02-core-interfaces.md` query.Builder invariants 5 and 7); and for every `unique` field a **partial unique index** `WHERE deleted_at IS NULL`.

**Trash-restore collision** *(Resolves EC-6)*: uniqueness binds only live rows, so trashing a record frees its unique values — a legitimate workflow (trash a record, create its replacement). Restoring checks the partial index the moment `deleted_at` clears: if a live row now holds the value, the restore fails with `409 Conflict` naming the colliding field and record, and the trashed row remains trashed. Restore never overwrites and never merges (BR-LIFE-5).

## The Live-Table/Revisions Contract (BR-LIFE-1, BR-LIFE-2)

- **Write.** Every create or update runs one transaction: write the live row (compare-and-set on `version` — BR-LIFE-7) and insert a `cms_revisions` row with the next `version_no` carrying the full field-slug-keyed JSONB snapshot.
- **Never-published record.** `status='draft'`; the live row holds the newest draft content so admin lists read live tables only.
- **Published record.** The live row holds the published content. Edits advance the live row's system columns (`version`, `updated_at`) via compare-and-set and insert a revision; the content columns remain frozen until publish (BR-LIFE-7). The admin edit view reads the newest revision; the public API reads the live row and never sees pending drafts.
- **Publish.** Copies the chosen revision's `data` into the live row, sets `status='published'`, moves the `published` flag to that revision (one atomic transaction).
- **Pending-draft detection.** A record has a pending draft when its newest `version_no` exceeds its published revision's `version_no` — derivable with one indexed query, no extra state on the live row.
- **Restore.** Appends a new revision copying the historical snapshot through the drift-mapping rules of `03-dynamic-schema.md` *(Resolves EC-5)*; history is append-only without exception.
- **Scheduling time source (V2).** `publish_at` comparisons in `jobs.Publisher` evaluate against database `now()`, never the process clock — scheduled-publish timing stays correct across restarts and host clock drift.

## Retention (BR-LIFE-8)

`jobs.Retention` ticks hourly:

1. Purges trash where `deleted_at < now() - CMS_TRASH_RETENTION_DAYS`, honoring FK RESTRICT — referenced records stay in trash and the skip is logged.
2. Prunes revisions beyond `CMS_REVISION_LIMIT` per record, oldest first, never touching the `published` revision or the newest revision.
3. Sweeps `cms_media` rows stuck in `pending` beyond 24 h, deleting the storage object first, then the row (BR-MEDIA-2).
4. Purges `cms_sessions` rows past their idle/absolute expiry window (BR-AUTH-5).
5. Purges `cms_reset_tokens` rows that are used or expired (BR-AUTH-13).
6. Purges `cms_refresh_tokens` rows that are rotated or revoked and older than 30 days.
7. Purges `cms_idempotency_keys` rows older than 24 h (`04-api-layer.md` §Idempotent Creates).

## Media Deletion

Deleting a `cms_media` row is destructive-gated the same way a collection drop is: re-authentication within the preceding 4-hour window (BR-AUTH-5) plus typed confirmation, per the BR-SCHEMA-7 pattern. While any `media` field on any record still references the row, its `ON DELETE RESTRICT` FK blocks the delete with `409 Conflict` naming the referencing record. On success the row is deleted first, then the storage object; the orphan sweep (BR-MEDIA-2) is the backstop for the crash window between those two steps, so a process death mid-delete cannot strand an orphaned object indefinitely.

## Erasure (V2, F-33)

GDPR-class erasure for end users lands in V2. Deleting a `cms_end_users` row is a hard delete of the row itself plus every `cms_refresh_tokens` row and every `cms_reset_tokens` row with `user_kind='end_user'` for that `user_id`, all in one transaction. Content the user authored is not deleted with them: `created_by` on affected `c_<slug>` rows and `cms_revisions` rows is rewritten to a fixed tombstone UUID, preserving referential integrity and the audit trail without retaining the identity link. `jobs.Retention` additionally gains a revision-redaction capability that can strip content-embedded PII from historical `cms_revisions.data` snapshots on request; this has documented limitations — it redacts known fields, not arbitrary free-text mentions of a user elsewhere in a document — and does not by itself guarantee complete erasure of PII an admin embedded as unstructured text.

## Admin Deletion

Deleting a `cms_users` row cascades its `cms_sessions` and `cms_reset_tokens` rows, so a removed admin retains no live session and no pending reset token. `created_by` references on `c_<slug>` rows, `cms_revisions`, and `cms_media` are left untouched by design — the UUID persists even after it no longer resolves to a `cms_users` row, preserving authorship history and the audit trail (contrast with end-user erasure above, which anonymizes `created_by` deliberately). Audit events (`08-observability.md`) store a snapshot of the acting admin's identity at the time of the action, so historical entries stay legible after the admin row is gone.

## Sizing Assumptions

The reference targets in `../REQUIREMENTS.md` (N-3, N-4) assume ≤100,000 live rows per collection, ≤50 revisions per record (the default cap), and JSONB snapshots ≤256 KB. Documents exceeding these bounds still function; the latency targets stop binding. Independent of these targets, `content.Document.Set` caps any single field value at 1 MiB — larger values fail with `422 validation_failed` (`02-core-interfaces.md`) — bounding how much a single oversized field can inflate a revision snapshot.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-5 | Snapshot format (field-slug-keyed JSONB) + append-only restore through drift mapping (Live-Table/Revisions Contract) |
| EC-6 | Partial unique indexes on live rows + 409-on-restore semantics (Collection Table Anatomy) |
