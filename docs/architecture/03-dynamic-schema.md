# 03 — Dynamic Schema Engine

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

`schema.Engine` is the only component that issues DDL. It converts validated schema-change requests into whitelisted DDL operations plus metadata updates (`cms_collections`, `cms_fields`), committed atomically. This document specifies the definition model, every whitelisted operation, the safe-conversion matrix, concurrency behavior, destructive-change gates, and drift handling. Physical table layouts live in `07-data-model.md`.

## Definition Model

A **collection** is `{slug, name, access_rules, search_config (V2)}`. A **field** is `{slug, type, config, position}` where `config` carries `required`, `unique`, `indexed`, `default`, relation target and cardinality, and numeric precision where applicable.

Validation at the admin API boundary (`httpapi/admin.validateSlug`, re-checked by the engine — BR-SCHEMA-2):

- Slugs match `^[a-z][a-z0-9_]{0,54}$`.
- The blocklist rejects Postgres reserved words, the seven system column names (BR-SCHEMA-8), and the prefixes `cms_`, `c_`, `cj_`.
- A collection holds at most 200 user fields — headroom under Postgres's 1600-column ceiling and a sanity bound on admin UI rendering.
- `required` enforcement is application-level (`content.Document.Set`): user columns are always nullable in DDL, so ADD COLUMN never fails against existing rows. `unique` enforcement is database-level via partial unique indexes (see `07-data-model.md`).

## Whitelisted Operations (BR-SCHEMA-4)

The engine executes exactly these operations; `schema.Engine.Apply` rejects anything else. Identifiers pass through `query.Builder.QuoteIdent` in every template (BR-SCHEMA-3).

| Operation | Validations | DDL template | Failure modes |
|---|---|---|---|
| CreateCollection | slug rules; slug unused | `CREATE TABLE "c_<slug>" (<system columns>)` + default indexes | slug collision → 409 |
| DropCollection | destructive gate; no inbound FKs | `DROP TABLE "c_<slug>"` | inbound relation exists → 409 listing referrers |
| RenameCollection | slug rules; new slug unused | `ALTER TABLE "c_<old>" RENAME TO "c_<new>"` | slug collision → 409 |
| AddField | slug rules; field cap; type valid | `ALTER TABLE ... ADD COLUMN "<slug>" <type>` (nullable) | cap exceeded → 422 |
| DropField | destructive gate | `ALTER TABLE ... DROP COLUMN "<slug>"` | — |
| RenameField | slug rules | `ALTER TABLE ... RENAME COLUMN "<old>" TO "<new>"` | slug collision → 409 |
| ChangeFieldType | safe-conversion matrix | `ALTER TABLE ... ALTER COLUMN ... TYPE <type> USING <cast>` | outside matrix → 422 (EC-3) |
| AddIndex / DropIndex | field exists | `CREATE INDEX` / `DROP INDEX` on `"c_<slug>"` | duplicate index → no-op |
| AddForeignKey / DropForeignKey | relation field; target exists | `ALTER TABLE ... ADD CONSTRAINT ... REFERENCES "c_<target>"(id) ON DELETE RESTRICT` | orphaned values present → 409 with count |

## Safe-Conversion Matrix (BR-SCHEMA-5)

The engine permits exactly two conversion classes; it rejects everything else with a remediation message naming the drop-and-recreate path. *(Resolves EC-3.)*

| From → To | Allowed | Cast |
|---|---|---|
| `number(p,s)` → `number(p′,s′)` where p′≥p and s′≥s | Yes | implicit |
| `text`, `number`, `boolean`, `datetime`, `json`, `richText` → `text` | Yes | `USING <col>::text` |
| `relation` → anything | No | FK semantics cannot cast |
| `media` → anything | No | object reference cannot cast |
| all other pairs | No | — |

Conversion to `text` of a `richText` or `json` field yields the raw JSON serialization; the admin UI warns before submission.

## Concurrency and Atomicity

Every schema change runs inside one transaction that first acquires `pg_advisory_xact_lock` on the single global schema-lock key. The DDL statement and the metadata writes commit or roll back together — Postgres DDL is transactional (BR-SCHEMA-6).

**Concurrent schema change vs. in-flight writes** *(Resolves EC-1)*: three mechanisms compose.

1. The advisory lock serializes schema changes against each other — two admins cannot interleave DDL.
2. Postgres's own `ACCESS EXCLUSIVE` lock on the altered table queues the DDL behind in-flight DML statements and queues new DML behind the DDL; no write ever sees a half-altered table. New reads also queue: every whitelisted ALTER TABLE takes ACCESS EXCLUSIVE — briefly for metadata-only changes, for the full rewrite duration on type changes — and ACCESS EXCLUSIVE conflicts with ACCESS SHARE.
3. The in-memory schema cache swaps atomically before the advisory lock releases (BR-RUNTIME-7), so no request planned after the change uses stale metadata. A request planned *before* a destructive change may still reference a dropped column and receives the standard error envelope; with a single-tenant admin population, this window is accepted and audited rather than prevented.

## Destructive Changes (BR-SCHEMA-7)

DropField and DropCollection demand (a) re-authentication within the preceding 4 hours (`middleware.RequireRecentAuth`) and (b) typed confirmation of the exact target slug in the request body. Dropping a field removes live column data; every prior value survives in `cms_revisions` JSONB snapshots, which store data by field slug at write time. Dropping a collection deletes its table, its `cms_fields` rows, and its entire `cms_revisions` history in one transaction; the typed confirmation states this explicitly. (Field drops retain revision data; collection drops do not — the asymmetry is deliberate.) *(Resolves EC-2.)*

## Collection Rename (EC-4)

`cms_revisions`, relation FKs, and record IDs reference collections by `cms_collections.id` (UUID), never by slug — a rename therefore touches exactly two things in one transaction: the `slug` column in `cms_collections` and the physical table name. History, relations, and revisions survive untouched. The observable consequence is that API paths (`/api/collections/<slug>/...`) change immediately; the response to a stale path is `404`, and V2 redirects (F-21) can bridge public-facing paths. The admin UI surfaces this consequence in the rename confirmation dialog. *(Resolves EC-4.)*

## Revision Restore After Schema Drift (EC-5)

Restore maps a historical JSONB snapshot onto the *current* schema; it never fails on drift and never rewrites history (BR-LIFE-1). Mapping rules, applied by `lifecycle.Service.Restore` before the result passes through `content.Document.Set`:

1. Field in snapshot **and** current schema, same type → value restores.
2. Field in snapshot **and** current schema, type changed since → the engine attempts the safe cast; on failure the field restores as `null` and the audit event records the skipped field.
3. Field in snapshot only (since dropped) → value ignored; audit event records it.
4. Field in current schema only (added since) → field takes its configured `default`, else `null`; `required` violations surface as validation errors that block the restore with a field-level message.

*(Resolves EC-5; `07-data-model.md` specifies the snapshot format.)*

## Schema Cache

`schema.Cache` loads all collection and field metadata at startup (BR-RUNTIME-3) into an immutable snapshot behind an atomic pointer. `schema.Engine.Apply` builds the successor snapshot inside the schema transaction and swaps it after commit, before releasing the advisory lock. Readers never lock; they dereference the current snapshot per request.

## FTS Interaction (V2) — EC-12

Each collection's `search_config` names the searchable fields and weights. V2 materializes search as a generated `tsvector` column plus GIN index on the collection table. Any schema change that affects search — editing `search_config`, dropping or type-changing a searchable field — regenerates the column and index inside the same advisory-locked transaction as the triggering change. The rebuild holds ACCESS EXCLUSIVE: reads and writes to that collection stall for its duration (~seconds at the 100k-row design point; the audit event records the duration — schedule heavy changes off-peak). Search results are never stale relative to committed schema and never reference dropped fields. *(Resolves EC-12.)*

**V2 Alternative: `CREATE INDEX CONCURRENTLY` with Reconciliation.** To avoid stalls during large-table rebuilds, an alternative strategy maintains the `tsvector` column through database triggers (one INSERT/UPDATE trigger per collection, dynamically created at schema load). Index creation uses `CREATE INDEX CONCURRENTLY` outside the schema transaction, with a subsequent reconciliation step to detect and repair any rows inserted during the index build. This trades synchronous lockout for bounded staleness (stale reads during the ~seconds-long index build) and an async reconciliation pass. This alternative is adopted only if rebuild stalls become material at real collection sizes — a documented escape hatch, not a configuration option; both strategies guarantee search results never reference dropped fields.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-1 | Advisory lock + Postgres table locking + pre-release cache swap (Concurrency and Atomicity) |
| EC-2 | Destructive gates + revision snapshots (Destructive Changes) |
| EC-3 | Safe-conversion matrix with 422 rejection (Safe-Conversion Matrix) |
| EC-4 | ID-based references; rename = slug + table rename in one transaction (Collection Rename) |
| EC-5 | Four-rule drift mapping on restore (Revision Restore After Schema Drift) |
| EC-12 | Synchronous tsvector regeneration under the schema lock (FTS Interaction); CONCURRENTLY alternative documented for large collections |
