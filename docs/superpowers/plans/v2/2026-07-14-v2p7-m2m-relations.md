# V2-P7 — Many-to-Many Relations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Relation fields gain `cardinality: "many"` backed by `cj_` join tables with a dedicated relation editor — F-23. The single riskiest V2 phase: it extends the schema engine, the write pipeline, and the read path at once.

**Architecture:** A `many` relation provisions no live-table column; it provisions `cj_<collection>_<field>(source_id CASCADE, target_id RESTRICT, position, PK(source,target))`. Memberships obey the exact content freeze contract: the join table always mirrors the **live row's** state (draft records: reconciled on save; published records: frozen — draft membership edits live only in revision snapshots until publish reconciles them). Snapshots store UUID arrays under the field slug, so revisions/restore/drift need no new machinery. Cardinality conversion one↔many is rejected in V2 (`ErrBadConversion`).

**Tech Stack:** Go 1.25, pgx/v5, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v2-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- SQL for collection tables **and join tables** only in `internal/query` (BR-SCHEMA-3 — `cj_` tables are collection storage); DDL only through `schema.Engine`'s closed whitelist (BR-SCHEMA-4).
- Every content write through `content.Document.Set` (BR-RBAC-5) — membership arrays are content.
- Errors only via `httpapi.WriteError`; no new runtime deps/env vars; no migration (all DDL is dynamic).
- Filtering/sorting on M2M fields → 422 in ALL scopes (spec §D7 — YAGNI, documented).
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V2-3 — run before Task 1)

- [ ] Confirm: `schema.FieldConfig{…RelationTarget uuid.UUID…}` and the V1 relation DDL (UUID FK column, `ON DELETE RESTRICT`) (V1-P3); the identifier-truncation helper (20-char + 8-hex-hash) — reused for join-table names; the rename op's transaction (EC-4) — it gains join-table renames; the field-drop typed-confirm path.
- [ ] Confirm `content.Set`'s relation validation (shape-only UUID; FK enforces existence — V1-P4 T3) — arrays follow the same philosophy.
- [ ] Confirm `lifecycle.Save`'s tx shape and the published-row content freeze (`SELECT … FOR UPDATE`, content columns only when draft — V1-P4 T4); Publish's copy step (V1-P4 T5) — reconciliation hooks ride these transactions.
- [ ] Confirm `ExpandRelations` (V1-P6 T5: one builder query per field, anonymous-strict targets, scalar null for invisible) — the `many` form returns arrays and **omits** invisible targets instead of nulling.
- [ ] Confirm the V2-P6 `TxHook` fires for these Saves unchanged (membership edits are `record.update` — no new event types).
- [ ] Confirm the purge 23503→`ErrReferenced` mapping and how it names the blocking reference — join-table FK constraint names must map back to (collection, field).

## File Structure

```
internal/schema/types.go / validate.go      Cardinality on FieldConfig (modify)
internal/schema/ddl.go / engine.go          join-table DDL trio + rename/drop cascade (modify)
internal/content/document.go                many-relation array validation (modify)
internal/query/junction.go                  join reconcile/read SQL (BR-SCHEMA-3)
internal/lifecycle/service.go               save/publish reconciliation (modify)
internal/httpapi/…records + expand          array serialization + expand-many (modify)
web/src/lib/components/widgets/RelationMany.svelte
web/src/routes/CollectionEdit.svelte        cardinality select (modify)
web/e2e/v2p7-m2m.spec.ts
docs/architecture/03-dynamic-schema.md      M2M anatomy, rename cascade, conversion stance (amend)
docs/architecture/07-data-model.md          cj_ layout (amend)
```

---

### Task 1: Schema engine — `Cardinality`, join-table DDL, rename/drop cascade

**Files:**
- Modify: `internal/schema/types.go` (`FieldConfig.Cardinality string` — `""`/`"one"` ≡ one, `"many"`; JSON `cardinality`), `internal/schema/validate.go`, `internal/schema/ddl.go`, `internal/schema/engine.go`, `docs/architecture/03-dynamic-schema.md`, `docs/architecture/07-data-model.md`
- Test: `internal/schema/m2m_test.go`

**Interfaces:**
- Produces:
  - Validation: `cardinality` legal only on `relation` fields (else the naming validation error); values ∈ {"", "one", "many"}.
  - `junctionTableName(colSlug, fieldSlug string) string` — `"cj_" + colSlug + "_" + fieldSlug` with the V1 truncation rule applied to the combined name when > 63 bytes.
  - DDL (whitelist additions), on add-field with `many`: `CREATE TABLE "<cj>" (source_id UUID NOT NULL REFERENCES "c_<src>"(id) ON DELETE CASCADE, target_id UUID NOT NULL REFERENCES "c_<tgt>"(id) ON DELETE RESTRICT, position INTEGER NOT NULL DEFAULT 0, PRIMARY KEY (source_id, target_id))` + `CREATE INDEX "ix_<cj>_target" ON "<cj>" (target_id)`; on field drop (typed-confirm, like any destructive op): `DROP TABLE "<cj>"`; on **collection or field rename**: `ALTER TABLE … RENAME`, index renamed, in the same advisory-locked tx (EC-4 extension); target-collection rename touches nothing (FK binds by OID).
  - Cardinality change on an existing field → `ErrBadConversion` (409/422 per V1 mapping) with remediation text "create a new field and migrate content".
  - Collection delete: inbound `many` references from other collections extend `ErrHasInboundRelations` (the engine's inbound scan now includes join-table FKs).
  - Snapshot: `Field` carries cardinality; the snapshot exposes `JunctionTable(col, field) string` for `internal/query`.

- [ ] **Step 1: Failing tests** — create posts + tags, add `posts.tags` relation many → `cj_posts_tags` exists with both FKs (`pg_constraint` delete rules asserted: CASCADE source / RESTRICT target) + target index; rename collection `posts→articles` → table is `cj_articles_tags`, FK intact, one tx; rename field → renamed; drop field with confirm → table gone; cardinality flip → `ErrBadConversion`; delete tags while referenced → `ErrHasInboundRelations` naming posts.tags; 63-byte truncation case (long slugs) produces a valid, stable name.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS.
- [ ] **Step 5: Amend docs** — 03: new "Many-to-Many (V2)" subsection: anatomy, naming+truncation, rename cascade, conversion stance, inbound-relation extension; 07: `cj_` row gains the layout and the "always mirrors the live row; drafts of published records defer to publish" contract sentence.
- [ ] **Step 6: Commit** — `git commit -m "feat: many-cardinality relations — join-table DDL with rename cascade (F-23)"`

---

### Task 2: `Document.Set` — membership array validation

**Files:**
- Modify: `internal/content/document.go`
- Test: extend `internal/content/document_test.go`

**Interfaces:**
- Produces: for a `relation` field with `many`: input must be an array (JSON `[]any`) of UUID strings — `[]` legal (clears), `null` legal (treated as `[]`); non-array → the standard per-type 422 error naming the field; > 500 entries → error naming the field and the cap (spec §D7); duplicate UUID → error naming the field; each element shape-validated as UUID (existence stays FK-enforced at write, matching the V1 one-relation philosophy). The validated `Document` carries the array under the field slug (order = array order = `position`).

- [ ] **Step 1: Failing tests** — table: valid 3-element array; empty; null→empty; scalar UUID → error ("expects an array"); 501 elements → cap error; duplicate → error; non-UUID element → error naming index.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: many-relation array validation in Document.Set (BR-RBAC-5)"`

---

### Task 3: Lifecycle — reconciliation with the freeze contract

**Files:**
- Create: `internal/query/junction.go`
- Modify: `internal/lifecycle/service.go` (Save, Publish, Purge error mapping)
- Test: `internal/query/junction_test.go`, extend lifecycle integration suite

**Interfaces:**
- Produces:
  - `internal/query/junction.go`: `query.JunctionReconcile(snap, col, field) (deleteSQL, insertSQL string)` — delete-then-insert diff form (`DELETE FROM "<cj>" WHERE source_id=$1` + batched `INSERT … (source_id, target_id, position) VALUES …` — full-rewrite reconcile is simplest-correct at ≤ 500 rows); `query.JunctionArrays(snap, col, field) string` — `SELECT source_id, array_agg(target_id ORDER BY position) FROM "<cj>" WHERE source_id = ANY($1) GROUP BY source_id`.
  - Save: **draft-status rows** reconcile the join table inside the Save tx from the validated array; **published rows** skip reconciliation (content freeze — membership edits live in the revision snapshot only). Revision snapshot: the array rides the merged snapshot under the field slug (existing snapshot machinery — the "live values" side reads via `JunctionArrays` for many fields).
  - Publish: after copying revision data into content columns, reconcile the join table from the revision's array (same tx). RestoreRevision inherits both (it runs Set→Save draft path). Trash: join rows untouched. Purge: source purge → CASCADE clears rows (no code); purging a *target* referenced in any join table → existing 23503→`ErrReferenced` with the constraint→(collection, field) name mapping extended to `cj_` constraints.
  - FK violation on reconcile INSERT (nonexistent/never-existed target) → maps to the 422 shape naming the field (parity with V1 one-relation writes; re-validate the V1 mapping location).

- [ ] **Step 1: Failing tests** — integration arc `TestF23_MembershipFreezeContract`: create draft post with tags [A,B] → join rows (A,0),(B,1); save [B,C] → rows (B,0),(C,1); publish → frozen: save [A] on the published row → join table STILL [B,C], newest revision snapshot says [A]; publish again → join [A]; restore the [B,C] revision → draft path → publish → [B,C] back. Plus: purge tag C while referenced → `ErrReferenced` naming posts.tags; purge the post → join rows gone; write with unknown target UUID → 422 naming the field; trash post → rows persist, restore intact.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: membership reconciliation under the content freeze contract (F-23)"`

---

### Task 4: Read path — arrays + expansion

**Files:**
- Modify: record serialization (admin + public), `internal/httpapi` expand path, `internal/query/junction.go` (expand query if needed beyond `JunctionArrays`)
- Test: extend public/admin read integration suites

**Interfaces:**
- Produces:
  - Unexpanded reads (single + list, admin + public): each `many` field serializes as a UUID array — **one `JunctionArrays` query per many-field per response page** (not per record); order by `position`.
  - `ExpandRelations` gains the many form: for each named many-field, collect page-wide target ids → ONE builder query (`ScopePublic` **anonymous-strict** form, exactly the V1 expansion posture) → replace each UUID with the embedded object; targets that are missing/trashed/draft are **omitted from the array** (spec §D7 — array shape, unlike the scalar `null` rule; the array's length is thereby visibility-dependent, documented in the 04 amendment inside this task).
  - Filtering/sorting: builder rejects filter/sort on many fields in every scope with the standard 422 naming the field.
- [ ] **Step 1: Failing tests** — list of 3 posts with shared tags → exactly 2 extra queries for [tags] (assert via pgx tracer or query counter seam — re-validate what V1 bench uses); expansion embeds published tags, omits a trashed one (array shrinks, no null holes), anonymous vs owner parity (anonymous-strict); `filter[tags][eq]=…` → 422 in public AND admin scopes; `?format=html` + expand composes (V2-P4 walk covers embedded targets).
- [ ] **Step 2:** FAIL. **Step 3:** implement (+ one-sentence 04 amendment: many-expansion omission rule). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: many-relation arrays and expansion on the read path (F-23)"`

---

### Task 5: UI — cardinality select + RelationMany editor, e2e

**Files:**
- Create: `web/src/lib/components/widgets/RelationMany.svelte`, `web/e2e/v2p7-m2m.spec.ts`
- Modify: `web/src/routes/CollectionEdit.svelte` (cardinality radio on relation fields, locked after creation), `web/src/routes/RecordEdit.svelte` (widget dispatch)

- [ ] **Step 1: Failing e2e** — schema half: add a relation field, pick "Many", save → field chip shows "many"; try changing cardinality on the saved field → control disabled with the remediation hint (ErrBadConversion surfaced proactively). Content half: tag a post with three tags via the picker (search-as-you-type against the target collection's admin list, chips reorderable by drag, remove ×), save, publish; public read (`page.request`) returns the UUID array in order; `?expand=tags` embeds objects; trash one tag in its collection → expanded array shrinks on next fetch.
- [ ] **Step 2: Implement** — `RelationMany.svelte`: chips + typeahead (reuses the V1 single-relation search endpoint), order = chip order, emits UUID array; cap message at 500. CollectionEdit: cardinality radio (One/Many) visible only on relation type at field creation; disabled with hint afterwards.
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: many-relation editor and cardinality picker (F-23)"`

---

### Task 6: Acceptance sweep

- [ ] `make test && make trace` green; waiver empty.
- [ ] Full e2e suite (V1 + v2p1–p7) green — special attention: V1 single-relation suites untouched (cardinality default preserves every V1 code path).
- [ ] `make bench` N-3/N-4 unchanged (bench collections have no many fields; assert the page-wide-query rule kept list reads at O(fields), not O(rows)).
- [ ] `git grep -n '"cj_' internal/ | grep -v 'query/\|schema/'` → empty (join SQL confined to its two legal homes).
- [ ] Doc amendments landed: 03 (anatomy/cascade/conversion), 07 (layout+contract), 04 (expansion omission rule).

## Self-Review Notes (execution-time attention)

- **Freeze contract for memberships** is THE design decision of this phase: join table mirrors the live row, drafts defer to publish. The tempting shortcut (reconcile on every save) silently publishes membership edits on published records — the Task 3 arc test exists to kill it.
- **Delete-then-insert reconcile** rewrites ≤ 500 rows per save; a diff-based UPSERT is premature at this cap (YAGNI; the arc test pins semantics, not strategy).
- **Omit-vs-null in expanded arrays** diverges from the scalar rule deliberately: `[a, null, c]` leaks existence + ordinal of an invisible record; omission does not. Spec §D7 pins it; the 04 amendment documents it.
- **`position` reuse on target restore:** restoring a trashed target re-lengthens arrays automatically (rows persisted) — covered implicitly by Task 3's trash case; no re-numbering needed since `position` gaps are harmless under `ORDER BY position`.
- **Junction naming collision** (`cj_posts_tags` from field `tags` vs a *collection* literally named `posts_tags` with field…) is prevented by the truncation-hash rule only probabilistically for long names; for short names, add the explicit engine check: a to-be-created junction name colliding with an existing table → `ErrSlugTaken` naming both origins (cheap, deterministic).
