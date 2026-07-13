# P3 Schema Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The runtime-DDL engine and schema cache: admins create/modify/drop collections and fields through `/api/admin/collections`, real `c_<slug>` tables materialize with the BR-SCHEMA-8 anatomy and default indexes, and UAC-1.1 passes via curl.

**Architecture:** Implements `03-dynamic-schema.md` in full (whitelist, conversion matrix, caps, destructive gates, EC-1/2/3/4) plus `07-data-model.md` §Collection Table Anatomy. `internal/schema` owns the `Engine` (sole DDL issuer — BR-SCHEMA-4) and the `Cache` (atomic snapshot, BR-RUNTIME-7). `app.Run` startup step 4 goes live. Admin handlers compose Engine + guards from P2.

**Tech Stack:** pgx v5 transactions, `pg_advisory_xact_lock(0x636D7301)`, sqlc over `cms_collections`/`cms_fields`, chi.

> **Authored before execution (D-V1-3 amendment):** re-validate against the codebase at execution start; docs win over drifted plan detail. Invoke the `schema-ddl-safety`, `sqlc-workflow`, and `api-conventions` skills before their tasks.

## Global Constraints

- DDL exists ONLY in `internal/schema` templates (+ `store/migrations`) — BR-SCHEMA-4/BR-SCHEMA-3; the whitelist is exactly the nine operations of 03 §Whitelisted Operations.
- Slugs: `^[a-z][a-z0-9_]{0,54}$`; blocklist = Postgres reserved words + the seven system column names (`id,status,version,created_at,updated_at,created_by,deleted_at`) + prefixes `cms_`, `c_`, `cj_` (BR-SCHEMA-2).
- Caps: ≤ **200 user fields** per collection, ≤ **500 collections**; violations → `422 validation_failed` (03 §Definition Model).
- Schema transaction: one tx, `pg_advisory_xact_lock(0x636D7301)` first, then `SET LOCAL lock_timeout = '5s'` and `SET LOCAL statement_timeout = '60s'` (09 §Timeouts); DDL + metadata commit or roll back together (BR-SCHEMA-6). Lock-timeout abort → `409 conflict` with a retry message (EC-1).
- Cache swap happens after commit, **before** the advisory lock releases (BR-RUNTIME-7). Readers dereference an atomic pointer, never lock.
- Type mapping (07): `text→TEXT`, `richText→JSONB`, `number→NUMERIC(p,s)` or bare `NUMERIC`, `boolean→BOOLEAN`, `datetime→TIMESTAMPTZ`, `media→UUID REFERENCES cms_media(id) ON DELETE RESTRICT` (inline — 03), `relation→UUID` (FK via AddForeignKey, `ON DELETE RESTRICT`), `json→JSONB`. User columns always nullable.
- System columns per BR-SCHEMA-8 exactly: `id UUID PK`, `status TEXT CHECK ('draft','published')`, `version BIGINT`, `created_at/updated_at TIMESTAMPTZ NOT NULL`, `created_by UUID`, `deleted_at TIMESTAMPTZ`.
- Default indexes per 07 §Indexes: partial `(status) WHERE deleted_at IS NULL`; `(created_by, id)` B-tree; per-`indexed` field composite `(field, id)`; per-`unique` field **partial unique** `WHERE deleted_at IS NULL`. Names `ix_<table>_<field>` with the 20-char+8-hash truncation rule past 63 bytes.
- Safe conversions ONLY (BR-SCHEMA-5): numeric widening (p′≥p ∧ s′≥s), declared→bare `NUMERIC`, any of `text,number,boolean,datetime,json,richText`→`text` via `USING <col>::text`; everything else → `422` naming the drop-and-recreate path (EC-3). Bare→declared numeric is narrowing → No.
- Destructive gates (BR-SCHEMA-7): DropField/DropCollection need `RequireRecentAuth` (4 h) + typed confirmation of the exact slug in the body. DropCollection deletes table + `cms_fields` rows + all `cms_revisions` for the collection in one tx; field drops keep revisions (EC-2 asymmetry).
- All references to collections use `id`, never slug (EC-4); rename = slug column + `ALTER TABLE RENAME` (+ dependent index renames) in one tx.
- Audit every operation: `schema.collection.create|rename|drop`, `schema.field.add|rename|drop|change_type|index|fk` with durations logged (08 §Schema-Change Visibility).
- Waiver shrink: `BR-SCHEMA-1 2 4 5 6 7 8`, `BR-RUNTIME-7`.
- Commits: plain messages, no trailer. Branch `main`.

## File Structure

```
internal/schema/types.go            Collection, Field, FieldType, Snapshot definitions
internal/schema/slug.go             validateSlug + blocklist (shared: engine re-check)
internal/schema/slug_test.go
internal/schema/cache.go            Cache: atomic snapshot load/swap (BR-RUNTIME-7)
internal/schema/cache_test.go
internal/schema/ddl.go              quoteIdent + the nine DDL templates + index naming
internal/schema/ddl_test.go
internal/schema/convert.go          safe-conversion matrix (BR-SCHEMA-5)
internal/schema/convert_test.go
internal/schema/engine.go           Engine.Apply: tx + advisory lock + metadata + swap
internal/schema/engine_integration_test.go
internal/store/queries/collections.sql   sqlc: collections + fields CRUD
internal/httpapi/admin_collections.go    handlers
internal/httpapi/admin_collections_integration_test.go
internal/app/app.go                 (modify) step 4 live: cache load; wire Engine
docs/trace-waivers.txt              (modify) shrink
```

---

### Task 1: Types, slug validation, blocklist

**Files:**
- Create: `internal/schema/types.go`, `internal/schema/slug.go`
- Test: `internal/schema/slug_test.go`

**Interfaces:**
- Produces:
  - `schema.FieldType` string enum: `TypeText "text"`, `TypeRichText "richText"`, `TypeNumber "number"`, `TypeBoolean "boolean"`, `TypeDatetime "datetime"`, `TypeMedia "media"`, `TypeRelation "relation"`, `TypeJSON "json"`.
  - `schema.FieldConfig{Required, Unique, Indexed bool; Default any; Precision, Scale *int; RelationTarget uuid.UUID; HideFrom, ReadOnlyFor []string}` (JSON-tagged; audiences validated in P6, carried now).
  - `schema.Field{ID uuid.UUID; Slug string; Type FieldType; Config FieldConfig; Position int}`.
  - `schema.Collection{ID uuid.UUID; Slug, Name string; AccessRules json.RawMessage; Fields []Field}`.
  - `schema.Snapshot{ Collections map[uuid.UUID]*Collection; BySlug map[string]uuid.UUID }` with `(s *Snapshot).Collection(slug string) (*Collection, bool)` and `(c *Collection).Field(slug string) (*Field, bool)`.
  - `schema.ValidateSlug(s string) error` — regex + blocklist; error text names the offending rule.
  - Caps constants: `MaxFields = 200`, `MaxCollections = 500`.
- Blocklist content: the seven system columns, `user`,`table`,`select`,`where`,`order`,`group`,`primary`,`references`,`constraint`,`index`,`check`,`default`,`grant`,`create`,`drop`,`alter`,`join`,`union`,`null`,`true`,`false`,`limit`,`offset`,`between`,`and`,`or`,`not`,`in`,`like`,`from`,`into`,`values` (full ANSI-reserved coverage; grow-only list), plus any slug beginning `cms_`/`c_`/`cj_`.

- [ ] **Step 1: failing tests** — table-driven `TestBR_SCHEMA_2_SlugRules`: accepts `posts`, `a`, `blog_posts_2`; rejects `Posts`, `1a`, `a-b`, 56-char, `cms_x`, `c_x`, `cj_x`, `select`, `deleted_at`, empty.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: schema types and slug validation (BR-SCHEMA-2)"`

---

### Task 2: DDL templates, identifier quoting, index naming

**Files:**
- Create: `internal/schema/ddl.go`
- Test: `internal/schema/ddl_test.go`

**Interfaces:**
- Produces (package-private, consumed by Engine):
  - `quoteIdent(s string) string` — `"` + doubled inner quotes + `"`; every identifier in every template passes through it (BR-SCHEMA-3).
  - `indexName(table, field string) string` — `ix_<table>_<field>`, truncation rule when > 63 bytes: each component to first 20 chars + `_` + 8-char hex of `sha256(table+"\x00"+field)`.
  - `columnDDL(f Field) (string, error)` — the type mapping; `media` emits the inline `REFERENCES cms_media(id) ON DELETE RESTRICT`; `number` renders `NUMERIC(p,s)` when both set, bare `NUMERIC` otherwise.
  - `createTableDDL(slug string, fields []Field) []string` — `CREATE TABLE "c_<slug>"` with the seven system columns + user columns, followed by the default index set: status partial, `(created_by,id)`, per-indexed `(field,id)`, per-unique partial unique.
  - `addColumnDDL`, `dropColumnDDL`, `renameColumnDDL`, `renameTableDDL(old, new string) []string` (includes dependent index renames computed from the field list), `changeTypeDDL(slug string, f Field, to FieldType, p, s *int) (string, error)` (`USING "<col>"::text` casts), `createIndexDDL`, `dropIndexDDL`, `addFKDDL(slug, field string, target string) string` (`ON DELETE RESTRICT`), `dropFKDDL`.

- [ ] **Step 1: failing tests** — `TestBR_SCHEMA_8_CreateTableCarriesSystemColumns` (rendered DDL contains all seven with exact types + CHECK), `TestBR_SCHEMA_3_IdentifiersQuoted` (a slug like `evil` renders `"c_evil"`; quoteIdent doubles quotes), `TestIndexNamingTruncation` (63-byte overflow yields the 20+8 form, deterministic), `TestDefaultIndexSet` (status partial + `(created_by,id)` present in every create), `TestMediaColumnInlineFK`.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: whitelisted DDL templates with quoting and index naming (BR-SCHEMA-3/8)"`

---

### Task 3: Safe-conversion matrix

**Files:**
- Create: `internal/schema/convert.go`
- Test: `internal/schema/convert_test.go`

**Interfaces:**
- Produces: `schema.CanConvert(from Field, to FieldType, toPrecision, toScale *int) error` — nil when allowed; otherwise an error whose text names the remediation ("create a new field and migrate content — this conversion cannot preserve data") for the 422 body (EC-3).

- [ ] **Step 1: failing tests** — the 03 §Safe-Conversion Matrix verbatim as a table: numeric widen yes / narrow no / declared→bare yes / bare→declared no; each of text,number,boolean,datetime,json,richText → text yes; relation→anything no; media→anything no; text→number no.
- [ ] **Step 2–4:** implement (pure function), PASS. **Step 5: Commit** — `git commit -m "feat: safe-conversion matrix (BR-SCHEMA-5, EC-3)"`

---

### Task 4: Schema cache

**Files:**
- Create: `internal/schema/cache.go`
- Create: `internal/store/queries/collections.sql` (+ `make generate`)
- Test: `internal/schema/cache_test.go` (unit, injected loader) + integration load case

**Interfaces:**
- sqlc queries: `ListCollections`, `ListFields` (ordered by position), `InsertCollection`, `UpdateCollectionSlug`, `DeleteCollection`, `InsertField`, `UpdateFieldSlug`, `UpdateFieldType`, `DeleteField`, `DeleteFieldsForCollection`, `CountCollections`, `CountFieldsForCollection`, `DeleteRevisionsForCollection` (used by DropCollection — table exists since 0001).
- Produces:
  - `schema.NewCache() *Cache`; `(*Cache).Load(ctx, q *store.Queries) error` — builds a `Snapshot` from both tables (startup step 4; failure aborts startup, N-11).
  - `(*Cache).Snapshot() *Snapshot` — atomic pointer dereference (`atomic.Pointer[Snapshot]`).
  - `(*Cache).swap(s *Snapshot)` — package-private; only `Engine.Apply` calls it (BR-RUNTIME-7).

- [ ] **Step 1: failing tests** — unit: `Snapshot()` returns what was swapped in, concurrent readers race-free (`go test -race`, readers loop during swaps); integration: `Load` on a DB seeded with two collections + fields reconstructs slugs, order, config round-trip (JSONB ↔ FieldConfig).
- [ ] **Step 2–4:** implement, `make generate`, PASS. **Step 5: Commit** — `git commit -m "feat: atomic schema cache with startup load (BR-RUNTIME-7)"`

---

### Task 5: Engine.Apply — the nine operations

**Files:**
- Create: `internal/schema/engine.go`
- Test: `internal/schema/engine_integration_test.go`

**Interfaces:**
- Consumes: pool, `store.Queries`, `Cache`, `audit.Recorder`, ddl/convert/slug internals.
- Produces:
  - `schema.NewEngine(pool *pgxpool.Pool, q *store.Queries, cache *Cache, rec *audit.Recorder, log *slog.Logger) *Engine`.
  - Operation request types (closed set — BR-SCHEMA-4): `CreateCollection{Slug, Name string; AccessRules json.RawMessage}`, `RenameCollection{ID uuid.UUID; NewSlug string}`, `DropCollection{ID uuid.UUID; ConfirmSlug string}`, `AddField{CollectionID uuid.UUID; Field Field}`, `RenameField{CollectionID, FieldID uuid.UUID; NewSlug string}`, `DropField{CollectionID, FieldID uuid.UUID; ConfirmSlug string}`, `ChangeFieldType{CollectionID, FieldID uuid.UUID; To FieldType; Precision, Scale *int}`, `AddIndex{CollectionID, FieldID uuid.UUID}`, `DropIndex{CollectionID, FieldID uuid.UUID}`, `AddForeignKey{CollectionID, FieldID uuid.UUID}`, `DropForeignKey{CollectionID, FieldID uuid.UUID}`.
  - `(*Engine).Apply(ctx context.Context, p access.Principal, op any) (*Snapshot, error)` — sequence: validate op (slug rules re-check, caps, matrix, confirm-slug match) → `pool.Begin` → `SELECT pg_advisory_xact_lock(0x636D7301)` → `SET LOCAL lock_timeout='5s'; SET LOCAL statement_timeout='60s'` → DDL exec(s) → metadata writes via `q.WithTx(tx)` → build successor snapshot → commit → `cache.swap(successor)` → audit emit with `duration_ms` → return. Lock-timeout SQLSTATE `55P03` maps to `ErrLockTimeout` (handlers → 409 retry message). Distinct sentinel errors: `ErrSlugTaken` (409), `ErrCapExceeded` (422), `ErrBadConversion` (422 w/ remediation), `ErrConfirmMismatch` (422), `ErrHasInboundRelations` (409 listing referrers), `ErrOrphanedValues` (409 with count).
  - DropCollection validation queries inbound relation fields (any field of `TypeRelation` whose `Config.RelationTarget == op.ID`) → `ErrHasInboundRelations`; inside the tx it deletes `cms_fields` rows, `cms_revisions` rows (`DeleteRevisionsForCollection`), the `cms_collections` row, then `DROP TABLE`.
  - AddForeignKey pre-checks orphans: `SELECT count(*) FROM "c_<slug>" WHERE "<field>" IS NOT NULL AND "<field>" NOT IN (SELECT id FROM "c_<target>")` → count > 0 → `ErrOrphanedValues`.

- [ ] **Step 1: failing integration tests** (each on a fresh testdb, migrated, cache loaded):
  - `TestBR_SCHEMA_6_DDLAndMetadataCommitAtomically` — force a failure after DDL (duplicate field slug on the second AddField in one Apply is not possible; instead: submit `ChangeFieldType` to an invalid target through a doctored op to bypass validation? No — never bypass. Use the documented failure: `RenameCollection` to a slug that collides at the DB unique constraint while passing pre-validation via a concurrent insert) — simpler deterministic approach: start a tx manually holding the advisory lock, fire `Apply` with a 100 ms ctx deadline, assert error AND no metadata row AND no table exists. Also assert the happy path: after CreateCollection, `c_<slug>` exists AND the `cms_collections` row exists.
  - `TestBR_SCHEMA_4_UnknownOpRejected` — `Apply(ctx, p, struct{}{})` errors without touching the DB.
  - `TestBR_SCHEMA_8_UAC_1_1_CreateCollectionAnatomy` — create with one `text` field → assert via `information_schema.columns` all seven system columns + the field; assert the four default indexes exist via `pg_indexes`.
  - `TestBR_SCHEMA_5_ChangeTypeOutsideMatrix422` — boolean→number rejected with remediation text; number(10,2)→number(12,4) succeeds and `information_schema` shows the new typmod.
  - `TestBR_SCHEMA_7_DropRequiresTypedConfirmation` — wrong `ConfirmSlug` → `ErrConfirmMismatch`, table survives; right slug → table gone, fields gone, revisions gone (seed one revision row first).
  - `TestBR_RUNTIME_7_SnapshotSwapsBeforeLockRelease` — after Apply returns, `cache.Snapshot()` already contains the new collection (observable ordering: Apply blocks until swap).
  - `TestEC4_RenameKeepsIDAndMovesTable` — rename; old table name gone, new present; collection ID unchanged; dependent index names renamed.
  - Cap tests: 201st field → `ErrCapExceeded`; (500-collection cap unit-tested against a stubbed `CountCollections` — do not create 500 real tables).
- [ ] **Step 2–4:** implement, PASS (`-race` on).
- [ ] **Step 5: Commit** — `git commit -m "feat: schema engine — whitelisted DDL under the schema advisory lock (BR-SCHEMA-4/5/6/7)"`

---

### Task 6: Admin collections API

**Files:**
- Create: `internal/httpapi/admin_collections.go`
- Modify: `internal/httpapi/router.go`
- Test: `internal/httpapi/admin_collections_integration_test.go`

**Interfaces:**
- Consumes: `schema.Engine`, `schema.Cache`, P2 guards (`RequireSession`, `RequireCSRF`, `RequireRecentAuth`, `RequireRole`).
- Produces routes under `/api/admin/collections` (all RequireSession; mutations RequireCSRF; role floor `admin` for schema changes — persona P-1/P-2; destructive add RequireRecentAuth):
  - `GET /` — list from snapshot. `POST /` — CreateCollection. `GET /{slug}` — one collection w/ fields.
  - `PUT /{slug}` — rename (body `{newSlug}`; response includes a `pathChange` notice — EC-4).
  - `DELETE /{slug}` — DropCollection (body `{confirm: "<slug>"}`; RequireRecentAuth).
  - `POST /{slug}/fields` — AddField. `PUT /{slug}/fields/{fieldSlug}` — rename or type change (body decides: `{newSlug}` vs `{type, precision, scale}`). `DELETE /{slug}/fields/{fieldSlug}` — DropField (typed confirm + RequireRecentAuth). `POST /{slug}/fields/{fieldSlug}/index` / `DELETE .../index` — AddIndex/DropIndex. `POST .../fk` / `DELETE .../fk` — AddForeignKey/DropForeignKey.
  - Error mapping: `ErrSlugTaken/ErrHasInboundRelations/ErrLockTimeout/ErrOrphanedValues → 409 conflict`; `ErrCapExceeded/ErrBadConversion/ErrConfirmMismatch/validation → 422` — all through `WriteError`.
- Slug→ID resolution goes through the snapshot; unknown slug → 404 (stale path after rename — EC-4).

- [ ] **Step 1: failing integration test** — with a logged-in super admin (P2 helpers): create → 201; `psql`-level check `c_<slug>` exists (UAC-1.1); create colliding slug → 409; drop without recent auth → 403 recent_auth detail; drop with typed confirm → 200 and table gone; field add/rename/type-change/drop round trip; 422 bodies name fields.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: admin collections API over the schema engine (UAC-1.1)"`

---

### Task 7: Startup step 4, smoke, waiver shrink

**Files:**
- Modify: `internal/app/app.go` (construct Cache+Engine; `cache.Load` between instance lock and step 5; failure aborts — N-11), `docs/trace-waivers.txt` (remove BR-SCHEMA-1,2,4,5,6,7,8 + BR-RUNTIME-7)
- Test: extend `internal/app/smoke_p2_integration_test.go` → rename `smoke_integration_test.go`; add the UAC-1.1 leg: setup → login → create collection with a text field → assert table.

- [ ] **Steps:** wire, smoke test green, `make test && make trace && make lint` green (trace drops by 8), curl smoke per run-and-verify step 3, commit — `git commit -m "feat: schema cache loads at startup; P3 smoke green (BR-RUNTIME-3 step 4)"`.

## Plan Self-Review Notes

- BR-SCHEMA-1 (naming) is covered by the anatomy/create tests asserting `c_<slug>` naming — name a `t.Run("BR-SCHEMA-1 ...")` inside `TestBR_SCHEMA_8_UAC_1_1_CreateCollectionAnatomy`.
- The 500-collection cap is unit-tested via stub, not 500 real DDL round-trips — deliberate test-cost decision; the code path is identical.
- `AccessRules` are stored/returned opaquely in P3 (validation is P6's §4 work); the admin API accepts the JSONB and persists it — flagged so P6 knows validation lands later, and the evaluator's fail-closed rule covers malformed interim rules.
- Atomicity failure-injection uses lock/deadline rather than validation bypass — never test by breaking the whitelist.
