# P4 Content Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The content backbone — `query.Builder` (the sole collection-SQL surface), `content.Document.Set` (the sole write path), and `lifecycle.Service` (live-table/revisions contract, optimistic locking, trash/restore/purge) — exposed through the admin records API with keyset cursors, so UAC-1.4 passes via curl.

**Architecture:** Implements 02's `query.Builder` (all seven invariants), `content.Document`, and `lifecycle.Service` contracts; 07's live-table/revisions contract and trash semantics (EC-5/EC-6); 04's pagination (BR-API-1) and admin filtering. squirrel enters the codebase here and only here.

**Tech Stack:** github.com/Masterminds/squirrel (confined to `internal/query` — depguard already enforces), pgx v5, sqlc for `cms_revisions` queries.

> **Authored before execution (D-V1-3 amendment):** re-validate at execution start; docs win. Invoke `query-builder-invariants`, `content-lifecycle-invariants`, `sqlc-workflow`, and `api-conventions` skills before their tasks.

## Global Constraints

- SQL for collection tables exists ONLY in `internal/query` (BR-SCHEMA-3); every identifier through `QuoteIdent`, identifiers never parameters, values never interpolated.
- The seven builder invariants of `02-core-interfaces.md` §query.Builder verbatim — callers cannot disable any.
- Every content write goes through `content.Document.Set` (BR-RBAC-5); unknown fields drop silently; `readOnlyFor` rejects with field errors; `required` enforced here; single-field cap **1 MiB** → field-level error.
- Live-table/revisions contract (07): every create/update = one tx writing live row (compare-and-set on `version`, BR-LIFE-7) + a `cms_revisions` row with next `version_no` and the full field-slug-keyed JSONB snapshot (BR-LIFE-1). Published rows freeze content columns until publish; edits advance `version`/`updated_at` only. Pending draft = newest `version_no` > published revision's.
- Publish (BR-LIFE-2): copy chosen revision's `data` into the live row, `status='published'`, move the `published` flag — one atomic tx. Publish/unpublish require role editor+ (BR-LIFE-3) — enforced in the service (not only routing).
- Trash: `deleted_at = now()` (BR-LIFE-4); every query appends `deleted_at IS NULL` except `ScopeTrash`. Restore clears it; unique collision → `409` naming field + colliding record; trashed row stays trashed (BR-LIFE-5, EC-6). Purge honors FK RESTRICT → `409`.
- RestoreRevision applies the four EC-5 drift rules, then passes through `Document.Set`; history is append-only.
- Pagination (BR-API-1): limit default 25 clamp 100; offset > 10,000 → `422`; mandatory `id` tiebreaker; admin keyset cursors = opaque base64 of (sort key, id); `cursor`+`offset` together → `422`.
- Admin/trash scopes: any schema field, full operator set `eq neq lt lte gt gte in contains`; `contains` = escaped-wildcard `ILIKE`, `text` fields only, else `422` naming the field (04 §Filtering).
- Record IDs UUIDv7 application-side; `version` starts at 1; timestamps app-side UTC.
- Collection queries run with `statement_timeout = 10s` (09) — set per-connection via pool config or `SET LOCAL` in the query path (execution picks the mechanism; document the choice in code).
- Audit: `content.record.create|update|publish|unpublish|trash|restore|purge|restore_revision` (BR-AUDIT-1).
- Waiver shrink: `BR-SCHEMA-3`, `BR-LIFE-1 2 3 4 5 7`, `BR-RBAC-5`, `BR-API-1`.
- Error responses only via `WriteError`; commits plain; branch `main`.

## File Structure

```
internal/access/decision.go        Decision, Predicate, FieldRuleSet types + FullAccess()
internal/query/builder.go          ForCollection/WithDecision/Where/Sort/Build
internal/query/paginate.go         Paginate, cursor encode/decode
internal/query/builder_test.go     SQL-string assertions (unit — no DB needed)
internal/query/paginate_test.go
internal/content/document.go       Document.Set
internal/content/document_test.go
internal/lifecycle/service.go      Save/Publish/Unpublish/Trash/Restore/RestoreRevision/Purge/Revisions
internal/lifecycle/drift.go        EC-5 four-rule mapping
internal/lifecycle/service_integration_test.go
internal/lifecycle/drift_test.go
internal/store/queries/revisions.sql
internal/httpapi/admin_records.go
internal/httpapi/admin_records_integration_test.go
internal/app/app.go                (modify) wire query/content/lifecycle
docs/trace-waivers.txt             (modify) shrink
```

---

### Task 1: Decision types and query.Builder core

**Files:**
- Create: `internal/access/decision.go`, `internal/query/builder.go`
- Test: `internal/query/builder_test.go`

**Interfaces:**
- Produces in `access`:
  - `access.PredicateKind` — `PredicateNone`, `PredicateOwnerOnly`; `access.Predicate{Kind PredicateKind; PrincipalID uuid.UUID}`.
  - `access.FieldRuleSet{Hidden map[string]bool; ReadOnly map[string]bool}` (resolved for the requesting audience).
  - `access.Decision{Allowed bool; Predicate Predicate; FieldRules FieldRuleSet; Principal Principal}` — carries the principal so the builder can compile owner-draft forms.
  - `access.FullAccess(p Principal) Decision` — implicit super_admin/admin grant (12 §3 step 1); the only Decision constructor until P6's evaluator.
- Produces in `query`:
  - `query.Scope` — `ScopeAdmin`, `ScopePublic`, `ScopeTrash`.
  - `query.Op` — `OpEq OpNeq OpLt OpLte OpGt OpGte OpIn OpContains`.
  - `query.ForCollection(snap *schema.Snapshot, col *schema.Collection, scope Scope, d access.Decision) (*Builder, error)` — errors on `!d.Allowed` (constructor requires the Decision — BR-RBAC-2 bypass unrepresentable).
  - `(*Builder).Where(field string, op Op, value any) *Builder` (accumulates; errors surface at Build) · `Sort(field string, desc bool) *Builder` · `Build() (sql string, args []any, err error)` · `BuildByID(id uuid.UUID) (sql string, args []any, err error)` (single-record fetch under the same scope filters) · `QuoteIdent(s string) string` (exported — schema templates reuse it per 03).
  - Scope compilation (invariants 2–4): non-trash → `"deleted_at" IS NULL`; `ScopePublic` → `"status" = 'published'`, relaxed to `("status" = 'published' OR "created_by" = $n)` when `d.Principal.Kind` is `end_user` or `api_key` (authenticated public — owner-draft, BR-API-2); `PredicateOwnerOnly` → `"created_by" = $n`.
  - Field gating (invariants 6–7): `Where`/`Sort` reject unknown-snapshot fields and `FieldRules.Hidden` fields; `ScopePublic` additionally requires `Config.Indexed || Config.Unique` and rejects `OpContains`; `OpContains` on non-`text` fields rejects everywhere; `OpIn` values must be a slice.

- [ ] **Step 1: failing unit tests** (SQL-string assertions, no DB):

```go
func TestBR_SCHEMA_3_IdentifiersQuotedValuesParameterized(t *testing.T) {
	b := mustBuilder(t, ScopeAdmin) // helper: snapshot w/ text field "title", indexed number "rank"
	sql, args, err := b.Where("title", OpEq, "x").Build()
	if err != nil { t.Fatal(err) }
	if !strings.Contains(sql, `"c_posts"`) || !strings.Contains(sql, `"title"`) {
		t.Errorf("identifiers must be quoted: %s", sql)
	}
	if strings.Contains(sql, "'x'") || len(args) == 0 {
		t.Errorf("values must be parameters, never interpolated: %s %v", sql, args)
	}
}

func TestBR_LIFE_4_TrashFilterAppendsExceptTrashScope(t *testing.T) {
	for _, tc := range []struct{ scope Scope; want bool }{
		{ScopeAdmin, true}, {ScopePublic, true}, {ScopeTrash, false},
	} {
		sql, _, _ := mustBuilder(t, tc.scope).Build()
		if got := strings.Contains(sql, `"deleted_at" IS NULL`); got != tc.want {
			t.Errorf("scope %v: deleted_at filter present=%v want %v", tc.scope, got, tc.want)
		}
	}
}

func TestBR_API_2_PublicScopeStatusAndOwnerDraft(t *testing.T) {
	t.Run("BR-API-2 anonymous strict form", func(t *testing.T) { /* status='published' only */ })
	t.Run("BR-API-2 authenticated end_user gets owner-draft OR", func(t *testing.T) { /* OR created_by = $n */ })
}

func TestBR_RBAC_6_PredicateCompiles(t *testing.T) { /* ownerOnly adds created_by = $n */ }
func TestInvariant6_UnknownAndHiddenFieldsRejected(t *testing.T) { /* Where("nope"), Where(hidden) → Build err */ }
func TestBR_API_4_PublicOperatorAndFieldGating(t *testing.T) {
	/* public: contains → err; unindexed field → err; admin: contains on text OK, on number → err */
}
```

- [ ] **Step 2: verify failure**, **Step 3: implement** with squirrel (`sq.StatementBuilder.PlaceholderFormat(sq.Dollar)`); `go get github.com/Masterminds/squirrel@latest`. `OpContains` renders `"<f>" ILIKE $n` with value `"%" + escapeLike(v) + "%"` (escape `%_\`). **Step 4: PASS + `make lint`** (depguard proves confinement). **Step 5: Commit** — `git commit -m "feat: query builder core — scopes, predicates, field gating (BR-SCHEMA-3, invariants 1-4/6-7)"`

---

### Task 2: Pagination and keyset cursors

**Files:**
- Create: `internal/query/paginate.go`
- Test: `internal/query/paginate_test.go`

**Interfaces:**
- Produces:
  - `query.Page{Limit int; Offset *int; Cursor string}`; `(*Builder).Paginate(p Page) *Builder`.
  - Behavior (invariant 5, BR-API-1): limit ≤ 0 → 25; > 100 → 100; offset > 10,000 → `ErrOffsetCeiling`; offset AND cursor → `ErrPageConflict`; every Build appends the `id` tiebreaker after the requested sort (`ORDER BY <sort> , "id" ASC`, or `"id" ASC` alone).
  - Cursor: `query.EncodeCursor(sortValue any, id uuid.UUID) string` (base64url of JSON `{"k":<v>,"id":"<uuid>"}`) / decode; `Paginate` with a cursor compiles the keyset condition `(<sort>, id) > ($k, $id)` (direction-aware: `<` for descending) and no OFFSET; `(*Builder).NextCursor(lastRow map[string]any) string` helper for handlers.
  - Sentinel errors exported for handler mapping to `422`.

- [ ] **Step 1: failing tests** — clamp table (0→25, 250→100), ceiling 10_001 → error, both-supplied → error, tiebreaker always present (`TestBR_API_1_...` umbrella with subtests), cursor round-trip, keyset SQL shape for asc and desc, cursor ignores OFFSET.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: capped offset + keyset cursor pagination (BR-API-1, EC-11)"`

---

### Task 3: content.Document.Set

**Files:**
- Create: `internal/content/document.go`
- Test: `internal/content/document_test.go`

**Interfaces:**
- Produces:
  - `content.FieldError{Field, Message string}` (marshals into `WriteError` details).
  - `content.Document{Values map[string]any}` — validated, ready for lifecycle.
  - `content.Set(snap *schema.Snapshot, col *schema.Collection, rules access.FieldRuleSet, input map[string]any) (Document, []FieldError)` (BR-RBAC-5).
  - Per-type validation: `text` string; `number` json.Number/float within declared precision-scale (reject NaN/Inf); `boolean` bool; `datetime` RFC3339 string → `time.Time`; `json` any marshalable; `richText` Tiptap shape — an object whose `type == "doc"` with `content` array (schema-constrained walk one level: nodes are objects with string `type`); `media`/`relation` UUID string. `media` finalized-ness is verified by a `MediaVerifier` seam wired in P7 — P4 passes `nil` (no media rows can exist before P7) and `Set` accepts shape-valid UUIDs when the verifier is nil. Signature therefore: `Set(snap, col, rules, input, mv MediaVerifier)` with `type MediaVerifier interface{ Finalized(ctx, uuid.UUID) bool }` — context threaded via the first arg becoming `SetCtx(ctx, ...)`? No: keep `Set(ctx context.Context, snap, col, rules, input map[string]any, mv MediaVerifier)`.
  - Rules: unknown input keys drop silently; `rules.ReadOnly[field]` → FieldError; `Config.Required` missing/nil on create → FieldError (update semantics: required checked only for keys present-or-absent-from-live handled by lifecycle passing merged input); any single value > 1 MiB (JSON-serialized length) → FieldError.

- [ ] **Step 1: failing tests** — `TestBR_RBAC_5_...` umbrella: unknown-field drop; readOnly reject; required reject; each type accept/reject pair; Tiptap doc-shape accept, `{"type":"hack"}` reject; 1 MiB+1 value reject (`strings.Repeat`); number precision overflow reject.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: content.Document.Set — the sole validated write path (BR-RBAC-5)"`

---

### Task 4: lifecycle.Save — live row + revision, optimistic locking

**Files:**
- Create: `internal/lifecycle/service.go` (Save + Revisions), `internal/store/queries/revisions.sql`
- Test: `internal/lifecycle/service_integration_test.go`

**Interfaces:**
- sqlc (`revisions.sql`): `InsertRevision`, `ListRevisions` (by collection_id+record_id, version_no desc, paged), `GetRevision`, `GetPublishedRevision`, `MaxVersionNo`, `SetRevisionPublished` (two statements: clear-then-set handled in tx by service), `DeleteRevisionsForRecord`.
- Produces:
  - `lifecycle.NewService(pool, q *store.Queries, cache *schema.Cache, rec *audit.Recorder, log *slog.Logger) *Service`.
  - `(*Service).Save(ctx, p access.Principal, col *schema.Collection, recordID *uuid.UUID, doc content.Document, expectedVersion int64) (Record, error)` — `recordID == nil` → create: UUIDv7, `version=1`, `status='draft'` (admin creates always draft — 12 §1), INSERT live row + revision 1, one tx (BR-LIFE-1). Update: `UPDATE "c_<slug>" SET <content-or-not>, version = version+1, updated_at = $t WHERE id = $id AND version = $expected AND deleted_at IS NULL` — zero rows → `ErrVersionConflict` (409, BR-LIFE-7). Content columns update only when live `status='draft'`; published rows advance system columns only (07 contract) — decided by reading the live row inside the tx (`SELECT ... FOR UPDATE`). Every path inserts the next-`version_no` revision with the **full merged snapshot** (live values overlaid with doc values — so revisions are complete field-slug-keyed documents).
  - `lifecycle.Record{ID uuid.UUID; Status string; Version int64; CreatedAt, UpdatedAt time.Time; CreatedBy uuid.UUID; Values map[string]any}`.
  - `(*Service).Revisions(ctx, col, recordID, page query.Page) ([]Revision, error)`; `lifecycle.Revision{VersionNo int64; Data map[string]any; Published bool; CreatedBy uuid.UUID; CreatedAt time.Time}`.
  - Live-row SQL for collection tables composes through `query.Builder` extensions: add `(*Builder).BuildInsert(values map[string]any, sys SystemValues) (sql, args, err)` and `BuildUpdate(id uuid.UUID, expectedVersion int64, values map[string]any, sys SystemValues) (sql, args, err)` to `internal/query` (BR-SCHEMA-3 keeps even INSERT/UPDATE inside query) — `SystemValues{Status string; Now time.Time; CreatedBy uuid.UUID; BumpVersionOnly bool}`.
- Interfaces consumed: `content.Document` (Task 3), snapshot (P3).

- [ ] **Step 1: failing integration tests** — create → live row exists w/ version 1 + revision 1 published=false; update draft → content changed, version 2, revision 2; stale expectedVersion → `ErrVersionConflict` (`TestBR_LIFE_7_StaleVersionConflicts` — UAC-1.4 first half); update of a published record (seed via direct publish in Task 5? no — seed by SQL flipping status... better: this test lands fully in Task 5 after Publish exists; here assert draft-path only + revision completeness `TestBR_LIFE_1_WriteIsLiveRowPlusRevisionAtomically` (kill mid-tx via ctx cancel → neither row)).
- [ ] **Step 2–4:** implement (+`make generate`), PASS. **Step 5: Commit** — `git commit -m "feat: lifecycle.Save — one-transaction live row + revision with optimistic locking (BR-LIFE-1/7)"`

---

### Task 5: Publish, Unpublish, Trash, Restore, RestoreRevision, Purge

**Files:**
- Create: `internal/lifecycle/drift.go`
- Modify: `internal/lifecycle/service.go`
- Test: `internal/lifecycle/drift_test.go`, extend `service_integration_test.go`

**Interfaces:**
- Produces:
  - `(*Service).Publish(ctx, p, col, recordID uuid.UUID, versionNo int64) error` — role floor: `p.Role.AtLeast(access.RoleEditor)` else `ErrPublishForbidden` (BR-LIFE-3; unpublish same check). One tx: copy revision `data` into live content columns, `status='published'`, `version+1`, clear old `published` flag, set new (the partial unique index enforces at-most-one — BR-LIFE-2).
  - `(*Service).Unpublish(ctx, p, col, recordID) error` — `status='draft'`, published flag stays on the revision that was live (history preserved; pending-draft math unaffected).
  - `(*Service).Trash(ctx, p, col, recordID) error` — sets `deleted_at`; `(*Service).Restore(ctx, p, col, recordID) error` — clears it; unique-collision surfaces as `ErrRestoreCollision{Field string; CollidingID uuid.UUID}` → 409 naming both (EC-6, BR-LIFE-5); detection: perform the UPDATE and map SQLSTATE 23505 back to the field via the violated index name (`ix_<table>_<field>`).
  - `(*Service).RestoreRevision(ctx, p, col, recordID, versionNo) (Record, error)` — loads snapshot data → `mapDrift(snapData, col) (mapped map[string]any, skipped []string, defaulted []string)` per EC-5 rules 1–4 → `content.Set` → append as a NEW revision + update live row (draft path) — never rewrites history; audit detail carries `skipped`/`defaulted`.
  - `(*Service).Purge(ctx, p, col, recordID) error` — hard DELETE live row + `DeleteRevisionsForRecord`; FK RESTRICT (relation/media references) maps SQLSTATE 23503 → `ErrReferenced` → 409.
  - `drift.go`: `mapDrift` — rule 1 same-type restore; rule 2 type-changed → try safe cast via `schema.CanConvert` + Go-side conversion, failure → null + skipped; rule 3 snapshot-only → ignored + recorded; rule 4 schema-only → `Config.Default` else null.
- [ ] **Step 1: failing tests** — unit `TestEC5_DriftMappingFourRules` (one case per rule); integration: `TestBR_LIFE_2_PublishCopiesRevisionAndMovesFlagAtomically` (publish rev 1 of a 3-revision record → live content = rev 1 data, flag on rev 1 only), `TestBR_LIFE_3_ContributorCannotPublish` (service-level deny), `TestBR_LIFE_5_RestoreCollision409` (trash A with unique value, create B with same value, restore A → ErrRestoreCollision naming field, A stays trashed — UAC-1.4 second half), `TestBR_LIFE_4_TrashedInvisibleOutsideTrashScope`, pending-draft detection (`newest version_no > published`), publish-then-edit freezes content (BR-LIFE-7 published-row path from Task 4).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: publish/trash/restore/purge lifecycle with drift-mapped revision restore (BR-LIFE-2/3/4/5)"`

---

### Task 6: Admin records API

**Files:**
- Create: `internal/httpapi/admin_records.go`
- Modify: `internal/httpapi/router.go`
- Test: `internal/httpapi/admin_records_integration_test.go`

**Interfaces:**
- Consumes: everything above + P2 guards; `access.FullAccess` for admin principals until P6 (editors/contributors get evaluator-backed Decisions only in P6 — P4 routes floor at `RequireRole(access.RoleEditor)` for mutations, `RequireRole(access.RoleViewer)` for reads, publish routes additionally rely on the service's BR-LIFE-3 check; the finer matrix arrives with P6 and this floor is noted as interim in code).
- Produces routes under `/api/admin/collections/{slug}/records` (RequireSession; mutations RequireCSRF; purge RequireRecentAuth):
  - `GET /` — list: `ParsePagination(r) (query.Page, error)` + `ParseFilters(r, ...)` helpers exported for P6 reuse; supports `filter[f][op]=v`, `sort`, `limit/offset/cursor`, `count=exact` (admin always authenticated), returns `meta.pagination` incl. `next_cursor` when keyset paging.
  - `POST /` — create (body caps: this route class gets the **5 MiB** `http.MaxBytesReader` — 04/09). `GET /{id}` — read (admin edit view reads newest revision per 07: response carries `record` (live) + `draft` (newest revision data) + `pendingDraft bool`).
  - `PUT /{id}` — update, body `{version, values}`; version-mismatch → 409 `conflict` `{details:[{field:"version",expected:<live>}]}` (04 example shape).
  - `POST /{id}/publish {versionNo}` · `POST /{id}/unpublish` · `DELETE /{id}` (trash) · `GET /?scope=trash` (trash listing via ScopeTrash) · `POST /{id}/restore` · `DELETE /{id}/purge` (RequireRecentAuth) · `GET /{id}/revisions` · `POST /{id}/revisions/{versionNo}/restore`.
- Error mapping: `ErrVersionConflict/ErrRestoreCollision/ErrReferenced → 409`; FieldErrors → 422 details; `ErrPublishForbidden → 403`; unknown slug/record → 404.

- [ ] **Step 1: failing integration test** — full UAC-1.4 arc via HTTP with a logged-in editor: create → publish → stale-version update → **409 conflict** → trash → absent from default list, present in trash scope → restore → back (test named `TestUAC_1_4_OptimisticLockAndTrashRestore` with BR-LIFE t.Runs); cursor paging walks 30 seeded records in 3 pages with stable order under concurrent insert; `contains` filter on text works admin-side; 5 MiB+ body → 413.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: admin records API — CRUD, publish, trash, revisions, keyset cursors (UAC-1.4)"`

---

### Task 7: Wiring, smoke, waiver shrink

**Files:**
- Modify: `internal/app/app.go` (construct lifecycle service; extend RouterConfig), `docs/trace-waivers.txt` (remove BR-SCHEMA-3, BR-LIFE-1,2,3,4,5,7, BR-RBAC-5, BR-API-1)
- Test: extend `internal/app/smoke_integration_test.go` — the P4 leg: create collection → create record → publish → stale-version 409 → trash → restore.

- [ ] **Steps:** wire, smoke green, `make test && make trace && make lint` green (waived count drops by 9), curl smoke per run-and-verify steps 4–6 (step 4's public fetch still 404s — public API is P6; smoke asserts the admin-side halves), commit — `git commit -m "feat: P4 smoke green — content core complete"`.

## Plan Self-Review Notes

- BR-LIFE-6 (publish-referencing-trashed serializes null) is deliberately NOT shrunk here — its observable half is public expansion (P6); the waiver file already assigns it P6.
- INSERT/UPDATE for collection tables live in `query.Builder` (`BuildInsert`/`BuildUpdate`) — an extension of 02's read-oriented contract, required by BR-SCHEMA-3's "only collection-SQL surface"; flagged for the execution-time review as an additive interface change (02 §Stability Rules: adding methods is minor).
- The interim `RequireRole` floors on admin records routes are replaced by evaluator Decisions in P6 — marked in code with a `// P6:` comment so the replacement is greppable.
- `MediaVerifier` is nil in P4 (no media rows can exist); P7 wires the real check — BR-MEDIA-3 stays waived to P7.
- run-and-verify smoke step 4 (public unauthenticated fetch) cannot pass until P6 — the P4 exit gate is UAC-1.4, not UAC-1.2.
