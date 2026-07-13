# V2-P3 — Redirects + SEO Metadata Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Admins manage 301/302 redirects resolved through a public lookup endpoint (F-21, UAC-2.4 redirect half), and records carry SEO metadata exposed distinctly in API responses (F-20).

**Architecture:** `cms_redirects` is a small system table with admin CRUD and one cacheable anonymous lookup. `seo JSONB NULL` joins the closed system-column set (second BR-SCHEMA-8 amendment; migration 0005 backfills) and rides the record pipeline end-to-end: validated closed keys in the content layer, snapshotted into revisions under the reserved `"$seo"` key, frozen-until-publish exactly like content, restored with revisions, exposed as a top-level `seo` sibling of `data`.

**Tech Stack:** Go 1.25, pgx/v5, sqlc, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v2-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code; docs win over drifted plan detail.
- SQL for collection tables only in `internal/query` (BR-SCHEMA-3); `cms_redirects` SQL lives in sqlc under `internal/store/`.
- Every content write goes through `content.Document.Set` (BR-RBAC-5) — SEO validation lives beside it, same gate.
- Errors only via `httpapi.WriteError` (BR-API-3); anonymous cacheable responses via `httpapi.WriteCached` (BR-API-5).
- No new runtime deps; no new env vars. Migration `0005`. Audit + vocabulary growth per spec B3.
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V2-3 — run before Task 1)

- [ ] Confirm: `content.Set(ctx, snap, col, rules, input map[string]any, mv MediaVerifier)` and `MediaVerifier{Finalized(ctx, uuid.UUID) bool}` (V1-P4 T3); `lifecycle.Save(ctx, p, col, recordID *uuid.UUID, doc content.Document, expectedVersion int64) (Record, error)` and the **full-merged-snapshot** revision rule (V1-P4 T4); `mapDrift` four-rule shape (V1-P4 T5); `httpapi.WriteCached(w, r, p, body []byte)` (V1-P6 T6); the V2-P2 state (migrations through `0004`, eight system columns, empty waiver).
- [ ] Confirm the record write/read envelope: where the request carries `data` and `version`, and how responses shape `{data, meta}` — the `seo` sibling attaches at those exact seams.
- [ ] Confirm the admin records list projection includes system columns (V2-P2 added `publish_at` there); `seo` follows the same projection route.

## File Structure

```
internal/store/migrations/0005_redirects_seo.sql   cms_redirects + seo backfill
internal/store/queries/redirects.sql               CRUD + lookup (sqlc)
internal/schema/ddl.go / validate.go               seo column+blocklist (modify)
internal/content/seo.go                            ValidateSEO closed-key validator
internal/lifecycle/service.go                      $seo snapshot/copy plumbing (modify)
internal/httpapi/redirect_handlers.go              admin CRUD + public lookup
web/src/routes/Redirects.svelte                    CRUD screen
web/src/routes/RecordEdit.svelte                   SEO panel (modify)
web/e2e/v2p3-redirects-seo.spec.ts                 UAC-2.4 redirect half + SEO round-trip
docs/BUSINESS_RULES.md                             BR-SCHEMA-8: nine columns (amend)
docs/architecture/07-data-model.md                 cms_redirects row + seo anatomy row (amend)
docs/architecture/03-dynamic-schema.md             blocklist note (amend, one line)
```

---

### Task 1: Migration 0005 — `cms_redirects` + `seo` column + amendments

**Files:**
- Create: `internal/store/migrations/0005_redirects_seo.sql`, `internal/store/queries/redirects.sql`
- Modify: `internal/schema/ddl.go` (`createTableDDL` + `seo JSONB`), `internal/schema/validate.go` (blocklist += `seo`), `docs/BUSINESS_RULES.md`, `docs/architecture/07-data-model.md`, `docs/architecture/03-dynamic-schema.md`
- Test: extend the anatomy test; `internal/store/redirects_test.go`

**Interfaces:**
- Produces: `cms_redirects` per spec §D3 (exact DDL below); every `c_<slug>` table has `seo JSONB NULL` (**nine** system columns); sqlc `CreateRedirect`, `UpdateRedirect`, `DeleteRedirect`, `ListRedirects`, `LookupRedirect(from_path)`.

- [ ] **Step 1: Failing tests** — anatomy test expects nine columns; store test: create/lookup/update/delete a redirect row; duplicate `from_path` → unique violation; `status_code=307` insert → CHECK violation.

- [ ] **Step 2:** FAIL (relation absent, column count 8).

- [ ] **Step 3: Migration** — `0005_redirects_seo.sql`:

```sql
-- 0005: cms_redirects (F-21) + seo system column (F-20, BR-SCHEMA-8 amendment).
CREATE TABLE cms_redirects (
    id          UUID PRIMARY KEY,
    from_path   TEXT NOT NULL UNIQUE CHECK (from_path LIKE '/%'),
    to_path     TEXT NOT NULL,
    status_code SMALLINT NOT NULL CHECK (status_code IN (301, 302)),
    created_at  TIMESTAMPTZ NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL
);

DO $$
DECLARE c RECORD;
BEGIN
  FOR c IN SELECT slug FROM cms_collections LOOP
    EXECUTE format('ALTER TABLE %I ADD COLUMN IF NOT EXISTS seo JSONB', 'c_' || c.slug);
  END LOOP;
END $$;
```

Queries (`redirects.sql`): standard sqlc CRUD; `-- name: LookupRedirect :one` is `SELECT to_path, status_code FROM cms_redirects WHERE from_path = $1`.

- [ ] **Step 4:** `createTableDDL` gains `seo JSONB` (no index — never filtered); blocklist gains `seo`; `make generate`.

- [ ] **Step 5:** PASS, including V1 rename/drift suites.

- [ ] **Step 6: Amend docs** — BR-SCHEMA-8: closed set text becomes "…, `deleted_at`, and from V2, `publish_at` and `seo`". 07: system-table matrix row for `cms_redirects` (columns + "exact-match lookup; no wildcards in V2"); anatomy row `| seo | JSONB | closed-key SEO document (F-20); NULL = none; rides the revision snapshot as "$seo" |`. 03: one line noting the blocklist includes the V2 system columns.

- [ ] **Step 7: Commit** — `git commit -m "feat: cms_redirects table and seo system column — migration 0005 (F-20, F-21)"`

---

### Task 2: SEO write/read path through the record pipeline

**Files:**
- Create: `internal/content/seo.go`
- Modify: `internal/lifecycle/service.go` (Save/Publish/RestoreRevision), `internal/query` (Save/read projections — re-validate exact builder files), record handlers (request/response shaping)
- Test: `internal/content/seo_test.go`, extend `internal/lifecycle/service_integration_test.go`

**Interfaces:**
- Consumes: the record write body (gains optional top-level `seo` object beside `data`), `MediaVerifier` (nil-tolerant, same rule as V1-P4).
- Produces:
  - `content.ValidateSEO(ctx, input map[string]any, mv MediaVerifier) (map[string]any, error)` — closed keys `title, description, canonical_url, og_title, og_description, og_image, og_type`; all optional; strings ≤ 512 chars (`description`/`og_description` ≤ 2048); `canonical_url` must parse as absolute http(s) URL; `og_image` must be a UUID string, and when `mv != nil`, `mv.Finalized` must hold (BR-MEDIA-3 parity); unknown key → error naming it (→ 422).
  - Lifecycle plumbing: the revision snapshot map gains `"$seo"` (the validated map, or absent when never set); the live `seo` column updates only when `status='draft'` (same freeze rule as content — 07 contract); Publish copies the revision's `$seo` into the live column; `mapDrift` passes `$seo` through untouched (rule 0: reserved keys are never drift-mapped); RestoreRevision therefore restores historical SEO with the content.
  - Read shaping: admin and public record responses expose `seo` as a top-level sibling of `data` (never inside it); list projections include it.

- [ ] **Step 1: Failing tests** — unit: `TestValidateSEOClosedKeys` (unknown key `keywords` → error naming it; oversize title; non-URL canonical; non-UUID og_image); integration: `TestSEORoundTripDraftPublishRestore` — create draft with seo → read shows sibling; publish; edit seo on the published record → public read still serves the published seo (freeze), admin newest revision shows the new one; publish again → public updated; restore revision 1 → seo reverts with it; `TestSEOAbsentIsNullNotEmpty` — record without seo returns `"seo": null`.
- [ ] **Step 2:** FAIL. **Step 3:** implement per Interfaces (handler: reject `seo` present-but-not-object → 422 naming `seo`; pass validated map to Save alongside the document — re-validate whether `content.Document` carries it as a field or Save takes a second param; prefer a `Document.SEO map[string]any` field so the BR-RBAC-5 single-gate shape holds). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: SEO metadata through the record pipeline (F-20)"`

---

### Task 3: Redirect service + admin CRUD API

**Files:**
- Create: `internal/httpapi/redirect_handlers.go`
- Modify: router registration, `internal/audit/actions.go` (+`routing.redirect.create|update|delete`)
- Test: `internal/httpapi/redirect_handlers_test.go`

**Interfaces:**
- Produces: `GET/POST /api/admin/redirects`, `PUT/DELETE /api/admin/redirects/{id}` — `RequireSession`+`RequireCSRF`, role floor `RoleAdmin` (routing changes are site-wide); body `{from_path, to_path, status_code}`; validation: `from_path` starts `/`, no whitespace, ≤ 1024 chars; `to_path` is a path starting `/` **or** absolute http(s) URL; `status_code ∈ {301,302}`; violations → 422 naming the field; duplicate `from_path` → 409 `conflict` naming it (map SQLSTATE 23505). Audit on every mutation with the new actions.

- [ ] **Step 1: Failing integration tests** — CRUD arc; duplicate → 409; `status_code=307` → 422 naming `status_code`; editor role → 403; each mutation emits its audit action (assert via `cms_audit_log` — V2-P1).
- [ ] **Step 2:** FAIL. **Step 3:** implement (thin handlers over sqlc; UUIDv7 ids app-generated like V1). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: admin redirect CRUD with audit (F-21)"`

---

### Task 4: Public lookup endpoint

**Files:**
- Modify: `internal/httpapi/redirect_handlers.go`, public router registration
- Test: extend `internal/httpapi/redirect_handlers_test.go`

**Interfaces:**
- Produces: `GET /api/v1/redirects/lookup?path=/old` — anonymous-allowed; hit → `200 {data:{to_path,status_code}}` via `WriteCached` (anonymous: `public, s-maxage=60, stale-while-revalidate=60` + ETag; credentialed: `no-store` — BR-API-5 verbatim); miss → `404 not_found`; missing/malformed `path` (no leading `/`) → 422 naming `path`. Exact match only (no normalization beyond nothing at all: the consuming site sends the path verbatim; trailing-slash variants are distinct rows — documented in 07's row from Task 1).

- [ ] **Step 1: Failing tests** — `TestUAC_2_4_RedirectLookupResolves` (seed a 301, lookup returns target+code); miss → 404; anonymous response carries the BR-API-5 header set and a 304 round-trip on `If-None-Match`; `path=old` (no slash) → 422.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: public redirect lookup endpoint (UAC-2.4)"`

---

### Task 5: UI — Redirects screen + SEO panel, e2e

**Files:**
- Create: `web/src/routes/Redirects.svelte`, `web/e2e/v2p3-redirects-seo.spec.ts`
- Modify: `web/src/lib/router.js` (+`#/redirects`), nav shell, `web/src/routes/RecordEdit.svelte` (SEO panel)

- [ ] **Step 1: Failing e2e** — redirects: create `/old-blog → /blog` 301 via UI, assert row in table; lookup via `page.request.get('/api/v1/redirects/lookup?path=/old-blog')` → 301 payload (UAC-2.4 half); duplicate create surfaces the 409 inline. SEO: open a record, fill title/description in the SEO panel, save, publish; assert the public read (`page.request`) carries the `seo` sibling; unknown-key injection is unreachable via UI (panel is closed-form) — assert the seven labeled inputs exist and nothing else.
- [ ] **Step 2: Implement** — `Redirects.svelte`: table + create/edit modal (three fields; status radio 301/302), delete with confirm; role-gated nav link (admin+). `RecordEdit.svelte`: collapsible "SEO" section with the seven inputs (og_image reuses the V1 media picker widget), values ride the existing save flow, `FieldErrors` surfaces 422s.
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: redirects screen and SEO editor panel (F-20, F-21)"`

---

### Task 6: Acceptance sweep

- [ ] `make test && make trace` green; waiver still empty.
- [ ] Full e2e suite green (V1 + v2p1 + v2p2 + v2p3).
- [ ] `make bench` unchanged (seo rides existing row reads; lookup is off the bench path).
- [ ] BR-SCHEMA-8 doc text says nine columns and the anatomy test asserts nine.
- [ ] `git grep -n "cms_redirects" internal/ | grep -v store/` → only handler references through sqlc.

## Self-Review Notes (execution-time attention)

- **SEO freeze semantics** mirror content exactly (draft-only live writes, publish copies) — this is the spec's "rides the record pipeline" made literal; simpler alternatives (always-live seo) would leak unpublished metadata to the public API.
- **`Document.SEO` field vs second Save param** — flagged as an interpretation decision; the field keeps one write gate (BR-RBAC-5) and one merged-snapshot rule.
- **No redirect-loop detection** (A→B, B→A): the lookup returns one hop and never follows; loops are the consuming site's semantics. Documented stance, not an oversight.
- **`routing.redirect.*`** starts a new vocabulary domain; the Task 3 vocabulary test (V2-P1) forces the registry update in the same commit.
