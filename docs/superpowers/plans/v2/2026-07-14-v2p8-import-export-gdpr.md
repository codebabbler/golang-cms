# V2-P8 — Import/Export + GDPR Erasure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** First-class schema export/import and content export, best-effort content import with documented limitations (F-25, UAC-2.2 export half), and GDPR-class end-user erasure with revision redaction (F-33, new BR-AUTH-15). **This phase closes the V2 gate.**

**Architecture:** Export serializes the snapshot with relation targets by *slug* (no database IDs), so a fresh instance reproduces identical definitions — the UAC-2.2 diff test is the contract. Import is strict all-or-nothing for schema (validate everything, then apply through `schema.Engine` in two passes: collections, then relation fields — cycles dissolve) and best-effort create-only for content (rows ride `Document.Set`/`Save` with a new additive `ForceID` option so exported IDs — and therefore relations — survive). Erasure is one transaction: hard-delete the principal + tokens, rewrite `created_by` to the tombstone UUID everywhere. Landing after V2-P7, export covers the final schema feature set including `cj_` memberships (D-V2-5).

**Tech Stack:** Go 1.25, pgx/v5, NDJSON streaming, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v2-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- SQL for collection tables only in `internal/query` (BR-SCHEMA-3) — content export streaming and the `created_by` rewrite both compile there.
- Every content write through `content.Document.Set` (BR-RBAC-5) — **content import has no bypass**; DDL only through `schema.Engine` (BR-SCHEMA-4) — **schema import has no bypass**.
- Errors only via `httpapi.WriteError`; no new runtime deps/env vars; no migration.
- Tombstone UUID: `00000000-0000-4000-8000-000000000000` (spec §D8, exact literal).
- Done = the **V2 gate**: UAC-2.1…2.4 end-to-end, waiver file empty, V1 gates green, bench unchanged.

## Re-Validation Preamble (D-V1-3 / D-V2-3 — run before Task 1)

- [ ] Confirm: the snapshot's collection/field shapes incl. V2 additions (`search_config`, `TrgmIndexed`, `Cardinality` — V2-P4/P5/P7); `Engine.Apply` op set (create collection, add field, set search config); `lifecycle.Save(ctx, p, col, recordID *uuid.UUID, doc, expectedVersion)` create path (UUIDv7 generation point — `ForceID` hooks there); `content.Set` + `MediaVerifier`; `RequireRecentAuth` + the typed-confirm pattern (V1: confirm value in the request body — re-validate exact field name); `ConfirmTyped` UI component; `jobs.Retention` duty structure; `cms_end_users` / `cms_refresh_tokens` / `cms_reset_tokens` layouts (07).
- [ ] Confirm every V2 phase P1–P7 landed (this plan exports what they built); migrations through `0008`.
- [ ] Confirm the display-resolution chain (`cms_users` → `cms_end_users` → `cms_api_keys`, 07) — the UI's "Deleted user" fallback hooks its miss case.

## File Structure

```
internal/export/schema.go        snapshot → portable document + import validation/apply
internal/export/content.go       NDJSON stream out; best-effort stream in
internal/query/export.go         full-row export SELECT; created_by rewrite (BR-SCHEMA-3)
internal/auth/erasure.go         one-tx erasure (BR-AUTH-15)
internal/lifecycle/service.go    SaveOpts{ForceID} additive option (modify)
internal/lifecycle/redact.go     RedactRevisions (+ jobs.Retention exposure)
internal/httpapi/export_handlers.go   export/import/erase/redact endpoints
internal/access/access.go        TombstoneUUID constant (modify)
web/src/routes/ImportExport.svelte
web/src/routes/EndUsers.svelte   Erase flow (modify)
web/src/routes/Revisions.svelte  Redact dialog (modify)
web/e2e/v2p8-export-gdpr.spec.ts
docs/BUSINESS_RULES.md           + BR-AUTH-15 (amend)
docs/architecture/07-data-model.md   Erasure section live (amend)
docs/architecture/09-deployment.md   export/import complement note live (amend)
```

---

### Task 1: Schema export + strict import

**Files:**
- Create: `internal/export/schema.go`, `internal/httpapi/export_handlers.go` (schema half)
- Modify: `internal/audit/actions.go` (+`schema.definition.export`, `schema.definition.import`)
- Test: `internal/export/schema_test.go`

**Interfaces:**
- Produces:
  - `export.SchemaDoc{FormatVersion int; ExportedAt time.Time; Collections []export.Collection}` — `Collection{Slug, Name string; AccessRules, SearchConfig json.RawMessage; Fields []export.Field}`; `Field{Slug, Type string; Config map[string]any; Position int}` — with `relationTarget` **replaced by the target's slug** (`relationTargetSlug`); no UUIDs anywhere in the document. `FormatVersion` is `1`.
  - `export.DumpSchema(snap) SchemaDoc`; `export.ImportSchema(ctx, eng *schema.Engine, snap, p access.Principal, doc SchemaDoc) error` — reject unknown `FormatVersion`; **validate the whole document first** (every slug legal, every `relationTargetSlug` resolvable within doc∪instance, every config well-formed), then: any collection slug already existing → `ErrImportConflict` naming it, nothing applied; apply pass 1: create all collections with their non-relation fields; pass 2: add relation fields (targets now exist — cycles dissolve); pass 3: set non-empty search configs. All through `Engine.Apply` (BR-SCHEMA-4 — the engine's per-op transactions apply; on a mid-import engine failure, report the applied prefix in the error: import is *documented* as non-transactional across ops, strict only at validation).
  - `GET /api/admin/export/schema` → the document (Content-Disposition attachment) — `RequireRole(admin)`; `POST /api/admin/import/schema` → 200 `{data:{collections_created}}` / 409 naming the conflict — `RequireRole(super_admin)` + `RequireRecentAuth`. Audit both.

- [ ] **Step 1: Failing tests** — `TestUAC_2_2_SchemaRoundTripIdentical` (integration, the gate test): build a rich schema on testdb A (text/richText/number/media/relation-one/relation-many fields, trgm flag, search config, access rules) → export → import into fresh testdb B → export B → **the two documents are deep-equal after zeroing `ExportedAt`**; cyclic relations (authors↔books) import via two passes; existing-slug conflict → nothing created (collection count unchanged); unknown formatVersion → error; recent-auth enforced (401/403 shape).
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: schema export/import with slug-keyed relations (F-25, UAC-2.2)"`

---

### Task 2: Content export — NDJSON stream

**Files:**
- Create: `internal/export/content.go` (export half), `internal/query/export.go`
- Modify: `internal/httpapi/export_handlers.go`, `internal/audit/actions.go` (+`content.collection.export`)
- Test: `internal/export/content_test.go`

**Interfaces:**
- Produces:
  - `query.ExportRows(snap, col) string` — full live-row SELECT (all user fields + `id,status,seo,publish_at,created_at,updated_at,created_by`; excludes `search_tsv`, `version`, `deleted_at` — trashed rows excluded by `deleted_at IS NULL`), ordered `id ASC`; many-relation fields hydrated page-wise via `query.JunctionArrays` (V2-P7).
  - `export.StreamContent(ctx, w io.Writer, deps…, col) (int, error)` — one JSON object per line `{id,status,seo,publish_at,created_at,updated_at,created_by,data:{…}}`, batches of 500 with `http.Flusher` flush per batch.
  - `GET /api/admin/export/content/{slug}` → `application/x-ndjson` attachment — `RequireRole(admin)`; audit with row count.
- [ ] **Step 1: Failing tests** — export 1 200 seeded rows (3 batches): line count exact, each line parses, m2m field is an ordered array, trashed row absent, `search_tsv`/`version` absent from lines; a draft AND a published row both present with correct `status`.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: NDJSON content export (F-25)"`

---

### Task 3: Content import — best-effort, create-only

**Files:**
- Modify: `internal/export/content.go` (import half), `internal/lifecycle/service.go` (`SaveOpts`), `internal/httpapi/export_handlers.go`, `internal/audit/actions.go` (+`content.collection.import`)
- Test: extend `internal/export/content_test.go`

**Interfaces:**
- Produces:
  - `lifecycle.SaveOpts{ForceID *uuid.UUID}` + variadic `Save(…, opts ...SaveOpts)` — additive; `ForceID` replaces the UUIDv7 generation on the create path only (update path ignores it). **Flagged additive-interface decision** (same class as V1's BuildInsert flag).
  - `export.ImportContent(ctx, deps…, col, r io.Reader, importer access.Principal) (Report, error)` — per line: parse (malformed → skipped `bad_json`); `id` already exists in the collection → skipped `id_exists`; single-pass fixups **before** `Document.Set`: relation values (scalar or array element) whose target row does not exist *at that moment* → nulled/dropped + recorded `relation_missing`; media UUIDs without a `cms_media` row → nulled + `media_missing`; then `Set` (validation failure → skipped `invalid: <msg>`) → `Save` with `ForceID` (creates as **draft**; `created_by` = importer — authorship not preserved) → exported `status=="published"` → publish through the normal lifecycle path (BR-LIFE invariants hold); exported `publish_at` ignored (schedules are not portable — documented). `Report{Imported int; Skipped []SkipEntry{ID, Reason string}; Fixups []FixupEntry{ID, Field, Reason string}}`.
  - `POST /api/admin/import/content/{slug}` (NDJSON body, 64 MiB cap for this route — re-validate against the V1 body-cap middleware's override seam) → 200 with the report — `RequireRole(super_admin)` + `RequireRecentAuth`; audit with counts only.
- [ ] **Step 1: Failing tests** — round-trip: export collection A (Task 2), import into a fresh same-schema instance → imported count = exported, IDs preserved, published rows published, relation arrays intact (targets imported earlier in the same stream by `id ASC` order… **assert the known limitation instead**: a forward-referencing row's relation is dropped + reported — best-effort is honest, not clever); collision skip; malformed line skip; media-missing fixup; report golden JSON; import through a viewer session → 403.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: best-effort create-only content import with fixup report (F-25)"`

---

### Task 4: GDPR erasure + revision redaction (BR-AUTH-15)

**Files:**
- Create: `internal/auth/erasure.go`, `internal/lifecycle/redact.go`
- Modify: `internal/access/access.go` (`TombstoneUUID`), `internal/query/export.go` (+`RewriteCreatedBy`), `internal/httpapi/export_handlers.go` (erase + redact routes), `internal/jobs/retention.go` (expose redaction routine), `internal/audit/actions.go` (+`auth.end_user.erase`, `content.revision.redact`), `docs/BUSINESS_RULES.md`, `docs/architecture/07-data-model.md`
- Test: `internal/auth/erasure_test.go`, `internal/lifecycle/redact_test.go`

**Interfaces:**
- Produces:
  - `access.TombstoneUUID = uuid.MustParse("00000000-0000-4000-8000-000000000000")`.
  - `query.RewriteCreatedBy(snap, col) string` — `UPDATE "c_<slug>" SET created_by = $1 WHERE created_by = $2` (BR-SCHEMA-3 home).
  - `auth.EraseEndUser(ctx, pool, q, snap, rec, p access.Principal, userID uuid.UUID) error` — **one transaction**: `DELETE FROM cms_end_users WHERE id=$1` (absent → `ErrNotFound`); `DELETE FROM cms_refresh_tokens WHERE end_user_id=$1` (all families — revocation by nonexistence); `DELETE FROM cms_reset_tokens WHERE user_kind='end_user' AND user_id=$1`; for every collection: `RewriteCreatedBy(tombstone, userID)`; `UPDATE cms_revisions SET created_by=$tomb WHERE created_by=$1`. Audit `auth.end_user.erase` with the **ID only** — no email (the identity link is what's being erased).
  - `POST /api/admin/end-users/{id}/erase` body `{"confirm":"<user email>"}` — `RequireRole(super_admin)` + `RequireRecentAuth` + `RequireCSRF`; confirm mismatch → 422 naming `confirm` (typed-confirm pattern, V1 parity).
  - `lifecycle.RedactRevisions(ctx, q, collectionID, recordID uuid.UUID, fields []string) (int, error)` — `UPDATE cms_revisions SET data = data - $3::text[] WHERE collection_id=$1 AND record_id=$2` returning affected count; field names validated against the snapshot (unknown → 422; `$seo` not addressable — field slugs only, documented limitation). Exposed on `jobs.Retention` as a callable routine (not a scheduled duty) and at `POST /api/admin/revisions/redact` `{collection_id, record_id, fields}` — super_admin + recent-auth; audit with field *names*.
  - `BUSINESS_RULES.md` +**BR-AUTH-15**: "End-user erasure is one transaction: the `cms_end_users` row, every refresh token, and every end-user reset token for that user are hard-deleted, and every `created_by` reference in collection tables and `cms_revisions` is rewritten to the tombstone UUID `00000000-0000-4000-8000-000000000000`. Live JWTs expire within 15 minutes and cannot refresh. Enforced in `auth.EraseEndUser`." 07 §Erasure: present tense.
- [ ] **Step 1: Failing tests** — `TestBR_AUTH_15_ErasureOneTransaction` (integration): end user with active refresh family, a reset token, records in two collections + revisions → erase → user row gone, zero tokens, every `created_by` = tombstone (both tables), all in one tx (kill-switch probe: force the revisions UPDATE to fail via an advisory-lock conflict seam → *everything* rolls back, user still present); refresh attempt after erasure → 401 (family gone); confirm-mismatch → 422, nothing deleted; `TestRevisionRedactionStripsFields`: 3 revisions with `email_body` → redact → key absent in all 3, other keys intact, published flag/count untouched (BR-AUDIT-3 untouched — redaction hits `cms_revisions`, never `cms_audit_log`); unknown field → 422.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS (+ `make trace`: BR-AUTH-15 traces, no waiver). **Step 5: Commit** — `git commit -m "feat: GDPR erasure with tombstone rewrite + revision redaction (BR-AUTH-15, F-33)"`

---

### Task 5: UI — Import/Export screen, Erase flow, Redact dialog, e2e

**Files:**
- Create: `web/src/routes/ImportExport.svelte`, `web/e2e/v2p8-export-gdpr.spec.ts`
- Modify: `web/src/lib/router.js` (+`#/import-export`), nav (super_admin section), `web/src/routes/EndUsers.svelte`, `web/src/routes/Revisions.svelte`

- [ ] **Step 1: Failing e2e** — ImportExport: download schema export (assert a `format_version:1` JSON downloads); re-import it unmodified → the 409 conflict surfaces naming the first colliding slug (strict mode proven in-browser; the fresh-instance identity diff is Task 1's Go test); content export downloads NDJSON with the seeded row count; content import of a 3-line fixture (one collision) → report renders "2 imported, 1 skipped (id_exists)". EndUsers: Erase opens `ConfirmTyped` demanding the email; wrong email → inline 422; correct → row gone, ReAuth modal honored (recent-auth). Revisions: Redact dialog (field checkboxes) → success toast with affected count; redacted field shows `—` in the compare view.
- [ ] **Step 2: Implement** — ImportExport: two cards (Schema / Content) with export buttons, import file-pickers, and a report table component; collection select for content. EndUsers/Revisions: the two flows above; all new nav/actions role-gated super_admin.
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: import/export screens, erase and redact flows (F-25, F-33)"`

---

### Task 6: V2 gate — programme acceptance sweep

- [ ] `make test` green; `make trace` green; `grep -c '^BR-' docs/trace-waivers.txt` → `0` (spec §C3).
- [ ] **UAC-2.1**: `web/e2e/v2p5-search.spec.ts` + `TestUAC_2_1_*` green (search across text + Tiptap; config change reindexes).
- [ ] **UAC-2.2**: `TestUAC_2_2_SchemaRoundTripIdentical` (Task 1) + `TestBR_HOOK_1_SignedDelivery`/`TestDeliveryWithin30sViaNudge` (V2-P6) green.
- [ ] **UAC-2.3**: `web/e2e/v2p2-schedule.spec.ts` + `TestBR_LIFE_9_DuePublishAndCatchUp` green.
- [ ] **UAC-2.4**: `web/e2e/v2p1-audit.spec.ts` + `TestUAC_2_4_RedirectLookupResolves` green.
- [ ] V1 gates stay green: full V1 e2e suite, `make bench` meets N-3/N-4 on the V1 baseline collections.
- [ ] Doc state: no architecture doc still describes a shipped V2 feature in future tense (`grep -n "V2 adds\|V2 may\|(V2)" docs/architecture/*.md` — every remaining hit must be about something this programme did NOT ship, i.e. nothing, or deliberately-retained escape-hatch text like 03's CONCURRENTLY alternative); `11-roadmap.md` V2 gate row satisfiable as written.
- [ ] Update `docs/architecture/11-roadmap.md` nothing — scopes are authoritative history; the gate table already defines V2 done. (Explicitly: no doc edit in this step beyond verifying.)

## Self-Review Notes (execution-time attention)

- **`ForceID` additive option** is this phase's only V1-interface touch — same review class as V1's flagged BuildInsert/BuildUpdate additions.
- **Import forward-references are dropped, not resolved** (single pass): two-pass content import (defer relation fixups) was rejected — it doubles memory and blurs the "best-effort with documented limitations" contract F-25 demands. The report makes the drop visible; the limitation is the feature.
- **Schema import non-transactional across engine ops** (strict validation, per-op transactions): a mid-import crash leaves a prefix — the error names it, and re-running yields the 409 (idempotent-ish recovery by conflict). Making the whole import one DDL transaction would hold the advisory lock across N table rewrites — worse. Documented in the 09 amendment.
- **Erasure leaves media objects**: files uploaded by the erased user remain (they're content, referenced by records with tombstoned authorship) — consistent with 07's "content the user authored is not deleted with them." Flag in the EndUsers confirm copy.
- **Redaction cannot strip `$seo`** or free-text mentions — 07's documented limitation, restated in the dialog's helper text.
