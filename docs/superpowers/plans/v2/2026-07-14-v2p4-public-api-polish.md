# V2-P4 — Public API Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The public API gains keyset cursors (F-27), `contains` filtering behind per-field `pg_trgm` GIN indexes (BR-API-4 restoration), and `?format=html` server-side rich-text rendering (F-19).

**Architecture:** Three independent read-path upgrades, no new tables. Migration 0006 enables the bundled `pg_trgm` extension; `schema.FieldConfig` gains `TrgmIndexed` and the engine's closed DDL whitelist gains the trgm GIN index pair. The V1 admin cursor mechanism is exposed publicly unchanged. A new pure `internal/richtext` package renders canonical Tiptap JSONB to HTML over a closed node/mark allowlist — stored documents never change. **This phase is API-only (spec D-V2-2 exception): no admin screens; the `TrgmIndexed` toggle's UI ships with V2-P5's search-config editor.**

**Tech Stack:** Go 1.25, pgx/v5, pg_trgm (bundled extension — N-9 holds).

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v2-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- SQL for collection tables only in `internal/query` (BR-SCHEMA-3); DDL only through `schema.Engine`'s closed whitelist (BR-SCHEMA-4) or the migration ledger (`0006`).
- Errors only via `httpapi.WriteError` (BR-API-3); cache contract via `WriteCached` (BR-API-5) — the ETag is over the exact body, so `?format=html` responses get distinct ETags for free.
- No new runtime deps (pg_trgm is bundled Postgres); no new env vars.
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V2-3 — run before Task 1)

- [ ] Confirm: `schema.FieldConfig{Required, Unique, Indexed bool; Default any; Precision, Scale *int; RelationTarget uuid.UUID; HideFrom, ReadOnlyFor []string}` (V1-P3 T1) — `TrgmIndexed` is additive; the engine's field create/update op surface and index-name helper (`ix_<table>_<field>` + truncation rule).
- [ ] Confirm the public grammar enforcement point: V1-P6 says **the builder enforces** ScopePublic's operator subset (no `contains`) and the public handler rejects `cursor` with 422 (F-27 was V2) — both gates change in this phase; find their exact tests (`TestBR_API_4_PublicGrammar`, cursor-rejection case) and update rather than duplicate.
- [ ] Confirm `ParsePagination` produces `query.Page{Limit; Offset *int; Cursor string}` and admin list handlers already emit `meta.pagination.next_cursor` — the public handler reuses both verbatim.
- [ ] Confirm how V1 Tiptap `image` nodes store their source (media UUID in `attrs` vs resolved URL) — Task 3's `image` rendering rule follows the stored shape.
- [ ] Migrations through `0005` applied; `0006` next.

## File Structure

```
internal/store/migrations/0006_pg_trgm.sql   CREATE EXTENSION
internal/schema/types.go                     FieldConfig.TrgmIndexed (modify)
internal/schema/ddl.go / engine.go           trgm index DDL pair in the whitelist (modify)
internal/query/builder.go                    ScopePublic contains gate (modify)
internal/richtext/render.go                  Tiptap JSONB → HTML
internal/richtext/render_test.go
internal/httpapi/public_records.go           cursor + format=html wiring (modify)
docs/BUSINESS_RULES.md                       BR-API-4 V2 text (amend)
docs/architecture/04-api-layer.md            grammar + pagination V2-live (amend)
docs/architecture/03-dynamic-schema.md       trgm note V2-live (amend)
```

---

### Task 1: `pg_trgm` extension + `TrgmIndexed` field config + engine DDL

**Files:**
- Create: `internal/store/migrations/0006_pg_trgm.sql` (one line: `CREATE EXTENSION IF NOT EXISTS pg_trgm;`)
- Modify: `internal/schema/types.go`, `internal/schema/ddl.go`, `internal/schema/engine.go` (whitelisted ops), `docs/architecture/03-dynamic-schema.md`
- Test: `internal/schema/trgm_test.go`

**Interfaces:**
- Produces: `FieldConfig.TrgmIndexed bool` (JSON `trgmIndexed`) — legal only on `text` fields (any other type → the engine's validation error path, 422 at the API); toggling it on/off is a field-update op whose DDL is `CREATE INDEX "ix_<table>_<field>_trgm" ON "<table>" USING gin ("<field>" gin_trgm_ops)` / `DROP INDEX`. Index name follows the V1 truncation rule with suffix `_trgm`. Field drop and collection rename carry the index (rename path renames `ix_<table>_*` generically — verify in the V1 rename suite).

- [ ] **Step 1: Failing tests** — `TestTrgmIndexProvisionedAndDropped` (integration): create collection with a `text` field `body`, update field config `trgmIndexed=true` → index exists in `pg_indexes` with `gin_trgm_ops`; toggle off → gone; `TestTrgmRejectedOnNonText`: `trgmIndexed` on a `number` field → engine validation error naming the field; rename suite still green (index renamed).
- [ ] **Step 2:** FAIL. **Step 3:** implement (config field + two DDL templates registered in the closed whitelist + validation clause). **Step 4:** PASS.
- [ ] **Step 5: Amend 03** — the `pg_trgm` paragraph moves from "V2 may restore" to present-tense: per-field `trgmIndexed` on `text` fields provisions the GIN index; the audit event records the build duration like every engine op.
- [ ] **Step 6: Commit** — `git commit -m "feat: per-field pg_trgm GIN indexes behind FieldConfig.TrgmIndexed (BR-API-4 groundwork)"`

---

### Task 2: Public `contains` gate + public keyset cursors

**Files:**
- Modify: `internal/query/builder.go` (ScopePublic operator gate), `internal/httpapi/public_records.go` (cursor un-rejection), `docs/BUSINESS_RULES.md` (BR-API-4), `docs/architecture/04-api-layer.md`
- Test: update `TestBR_API_4_PublicGrammar`; extend the public list integration suite

**Interfaces:**
- Produces:
  - Builder: `ScopePublic` accepts `contains` **iff** the target field is `text` AND `Config.TrgmIndexed`; the compiled SQL is the existing escaped-`ILIKE` form (identical to admin scope — the trgm GIN index serves it); any other public `contains` → the existing 422-mapped builder error naming the field. Admin/trash scopes unchanged.
  - Public handler: `cursor` param now flows through `ParsePagination` exactly as admin lists (V1's public 422-on-cursor case deleted); `cursor`+`offset` together → 422 (existing rule); responses emit `meta.pagination.next_cursor`; anonymous cursor pages remain cacheable (`WriteCached` unchanged — the cursor is in the URL, so shared caches key it naturally).

- [ ] **Step 1: Failing tests** — `TestBR_API_4_PublicGrammar` gains subtests: `contains` on a trgm-indexed text field → 200 with matching rows; on a non-trgm text field → 422 naming the field; on a trgm-indexed field via **anonymous** request → 200 (cacheable, header set asserted); `TestF27_PublicKeysetCursor` (integration): seed 30 published records, walk 3 cursor pages of 10 with no overlap/gap under a concurrent insert; `cursor`+`offset` → 422; invalid cursor → 422.
- [ ] **Step 2:** FAIL. **Step 3:** implement (one gate clause in the builder; delete the handler rejection). **Step 4:** PASS.
- [ ] **Step 5: Amend docs** — BR-API-4 text: "`contains` is public only on `text` fields with `trgmIndexed`; admin- and trash-scope unrestricted (V1 rule superseded by the V2 restoration this rule anticipated)." 04 §Filter grammar + §Pagination: mark the V2 sentences live (public cursor exposed, contains restored behind trgm). `make trace` green (BR-API-4's test name still traces).
- [ ] **Step 6: Commit** — `git commit -m "feat: public contains behind trgm indexes + public keyset cursors (BR-API-4, F-27)"`

---

### Task 3: `internal/richtext` — Tiptap JSONB → HTML renderer

**Files:**
- Create: `internal/richtext/render.go`, `internal/richtext/render_test.go`

**Interfaces:**
- Produces: `richtext.RenderHTML(doc map[string]any) (string, error)` — pure function, no I/O. Closed **node** allowlist: `doc`, `paragraph`→`<p>`, `heading` (attrs.level 1–6, clamp else 6)→`<h?>`, `text`, `bulletList`→`<ul>`, `orderedList`→`<ol>`, `listItem`→`<li>`, `blockquote`→`<blockquote>`, `codeBlock`→`<pre><code>`, `hardBreak`→`<br>`, `image` (per the re-validated stored shape; `src` must be http(s) else the node renders nothing)→`<img src alt loading="lazy">`. Closed **mark** allowlist on text nodes: `bold`→`<strong>`, `italic`→`<em>`, `link` (attrs.href http(s) only, else the mark is dropped)→`<a href rel="noopener nofollow">`. Unknown node type → render its `content` children only (text survives, structure dropped); unknown mark → dropped. ALL text and attribute values pass `html.EscapeString`. A non-`doc` root or non-object input → error (handler maps to 500 `internal` — stored canon is engine-validated, so this indicates corruption, not user error).

- [ ] **Step 1: Failing table-driven tests** — one case per node and mark; nesting (list in blockquote); mark stacking order deterministic (`<strong><em>` outer-to-inner by array order); XSS corpus: `<script>` as text is escaped, `javascript:` href drops the link mark, `onerror` attr never emitted (only the fixed attribute set is written); unknown node `iframe` yields its text children only; deep nesting (500 levels) returns without stack overflow (iterative or depth-capped at 200 → error).
- [ ] **Step 2:** FAIL. **Step 3:** implement with `strings.Builder` (≈150 lines). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: richtext HTML renderer over closed Tiptap allowlist (F-19)"`

---

### Task 4: `?format=html` on public reads

**Files:**
- Modify: `internal/httpapi/public_records.go` (single + list + expansion serialization), `docs/architecture/04-api-layer.md` (mark F-19 live)
- Test: extend the public read integration suite

**Interfaces:**
- Produces: `?format=html` on `GET /api/v1/collections/{slug}/records[/{id}]` — every `richText` field value in the response (top-level records AND expanded relation targets) is replaced by its rendered HTML string; `format=json` or absent → canonical JSONB unchanged; any other value → 422 naming `format`. Admin routes never accept it (admin edits canon). Stored data untouched (assert: re-read as json after an html read → identical JSONB).

- [ ] **Step 1: Failing tests** — `TestF19_FormatHTMLRendersRichText` (integration): publish a record with a Tiptap body → `?format=html` returns `<p>…</p>` string in the field, `?format=json` returns the object, second json read byte-identical (canonical untouched); expanded relation target's richText also rendered; `format=xml` → 422; anonymous html read carries BR-API-5 headers and its ETag differs from the json read's (bodies differ).
- [ ] **Step 2:** FAIL. **Step 3:** implement — a serialization-stage walk over the response maps: for each field whose schema type is `richText`, `richtext.RenderHTML`; render error → 500 `internal` (log the record id, never the document body). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: ?format=html public rendering (F-19)"`

---

### Task 5: Acceptance sweep

- [ ] `make test && make trace` green; waiver still empty.
- [ ] Full e2e suite green (no new screens — V1 + v2p1–p3 specs must not regress; the public API changes are additive).
- [ ] `make bench` unchanged on N-3/N-4 (json path untouched; html rendering is opt-in per request).
- [ ] Grep: `git grep -n "ILIKE" internal/ | grep -v query/` → empty (contains compilation stayed inside the builder).
- [ ] Doc amendments landed: BR-API-4 V2 text, 03 trgm live, 04 pagination/grammar/F-19 live.

## Self-Review Notes (execution-time attention)

- **Rank of the three features:** independent — execute tasks in any order except 4-after-3 and 2-after-1.
- **`contains` cacheability:** anonymous trgm queries are cacheable like any public read; no extra invalidation concern (same 60 s window).
- **HTML in JSON:** F-19 renders per-field HTML strings inside the JSON envelope (the response stays `application/json`); a whole-page HTML response was rejected — consumers embed fields, not pages. Flagged as the interpretation of "`?format=html` rich-text rendering".
- **Renderer hardening is the review focus:** the XSS corpus in Task 3 is the security gate; reviewers should try to extend it, not shrink it.
- **`TrgmIndexed` UI** intentionally absent here — lands beside V2-P5's search-config editor (same field-settings surface), keeping this phase API-only per spec D-V2-2.
