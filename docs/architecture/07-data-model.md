# 07 — Data Model

**Version:** 1.0 · **Last Updated:** 2026-07-07 · **Owner:** Miraj Aryal

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

V2 adds `cms_audit_log` (append-only — BR-AUDIT-3), `cms_redirects`, and `cms_webhooks`. V3 adds `cms_carts` and `cms_orders`. Their layouts are specified in their delivery cycles; their names are reserved now.

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

User-field storage follows the field-type reference: `text→TEXT`, `richText→JSONB`, `number→NUMERIC`, `boolean→BOOLEAN`, `datetime→TIMESTAMPTZ`, `media→UUID` FK to `cms_media` plus the delivery URL resolved at read time, `relation→UUID` FK to the target's `id` with `ON DELETE RESTRICT`, `json→JSONB`. User columns are always nullable in DDL; `required` enforcement lives in `content.Document.Set` (`03-dynamic-schema.md`).

**Indexes.** PK on `id`; a partial index on `(status) WHERE deleted_at IS NULL` serving the public published-only scope (BR-API-2); a B-tree per `indexed` field; and for every `unique` field a **partial unique index** `WHERE deleted_at IS NULL`.

**Trash-restore collision** *(Resolves EC-6)*: uniqueness binds only live rows, so trashing a record frees its unique values — a legitimate workflow (trash a record, create its replacement). Restoring checks the partial index the moment `deleted_at` clears: if a live row now holds the value, the restore fails with `409 Conflict` naming the colliding field and record, and the trashed row remains trashed. Restore never overwrites and never merges (BR-LIFE-5).

## The Live-Table/Revisions Contract (BR-LIFE-1, BR-LIFE-2)

- **Write.** Every create or update runs one transaction: write the live row (compare-and-set on `version` — BR-LIFE-7) and insert a `cms_revisions` row with the next `version_no` carrying the full field-slug-keyed JSONB snapshot.
- **Never-published record.** `status='draft'`; the live row holds the newest draft content so admin lists read live tables only.
- **Published record.** The live row holds the published content. Edits insert revisions only — the live row does not change — creating a *pending draft*. The admin edit view reads the newest revision; the public API reads the live row and never sees pending drafts.
- **Publish.** Copies the chosen revision's `data` into the live row, sets `status='published'`, moves the `published` flag to that revision (one atomic transaction).
- **Pending-draft detection.** A record has a pending draft when its newest `version_no` exceeds its published revision's `version_no` — derivable with one indexed query, no extra state on the live row.
- **Restore.** Appends a new revision copying the historical snapshot through the drift-mapping rules of `03-dynamic-schema.md` *(Resolves EC-5)*; history is append-only without exception.

## Retention (BR-LIFE-8)

`jobs.Retention` ticks hourly:

1. Purges trash where `deleted_at < now() - CMS_TRASH_RETENTION_DAYS`, honoring FK RESTRICT — referenced records stay in trash and the skip is logged.
2. Prunes revisions beyond `CMS_REVISION_LIMIT` per record, oldest first, never touching the `published` revision or the newest revision.
3. Sweeps `cms_media` rows stuck in `pending` beyond 24 h, deleting the storage object first, then the row (BR-MEDIA-2).

## Sizing Assumptions

The reference targets in `../REQUIREMENTS.md` (N-3, N-4) assume ≤100,000 live rows per collection, ≤50 revisions per record (the default cap), and JSONB snapshots ≤256 KB. Documents exceeding these bounds still function; the latency targets stop binding.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-5 | Snapshot format (field-slug-keyed JSONB) + append-only restore through drift mapping (Live-Table/Revisions Contract) |
| EC-6 | Partial unique indexes on live rows + 409-on-restore semantics (Collection Table Anatomy) |
