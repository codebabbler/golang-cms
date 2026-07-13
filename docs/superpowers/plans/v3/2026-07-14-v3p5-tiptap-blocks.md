# V3-P5 — Tiptap Blocks + V3 Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Editors embed product and video references in rich text; the public API resolves them to structured data, and a deleted target resolves `null` without breaking the post (UAC-3.3, F-31). **This phase closes the V3 gate — and with it, the whole three-version programme.**

**Architecture:** Two leaf nodes in the canonical Tiptap JSONB — `productBlock{collectionId, recordId}` and `videoBlock{collectionId, recordId, field}` — join `content.Set`'s richText allowlist with shape-only attr validation (blocks must survive later deletions, so existence is never checked at write). Resolution happens at public read time in the serialization walk: one batched anonymous-strict query per block type per page, `"resolved": null` for anything invisible. HTML rendering emits attribute-only `<figure>` placeholders (the renderer stays pure).

**Tech Stack:** Go 1.25, Tiptap 2 custom node extensions, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v3-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- Named extension points only (spec C5): `content.Set` richText allowlist and `richtext.RenderHTML` allowlist — no other V1/V2 contract is touched.
- Resolution posture = the ExpandRelations posture: anonymous-strict, published-only, hidden fields stripped per the target's rules.
- Errors via `WriteError`; no migration; no new env; vocabulary unchanged (blocks are content, not mutations).
- Done = the **V3 gate**: UAC-3.1…3.3 (incl. both manual steps), V1+V2 gates green, waiver empty.

## Re-Validation Preamble (D-V1-3 / D-V3-2 — run before Task 1)

- [ ] Confirm `content.Set`'s richText walk (V1-P4 T3: schema-constrained walk, nodes are objects with string `type`) and where its node allowlist lives — V2-P4's renderer and this phase's validation must share ONE allowlist source if execution finds them drifting (targeted improvement, in scope).
- [ ] Confirm the V2-P4 serialization walk seam (`format=html` field replacement) and V3 additions to it (V2-P7 expand-many, V3-P4 video objects) — block resolution rides the same walk.
- [ ] Confirm `richtext.RenderHTML`'s allowlist table and unknown-node rule (V2-P4 T3).
- [ ] Confirm the builder's anonymous-strict form used by `ExpandRelations` (V1-P6 T5) and `Collection.Commerce` on the snapshot (V3-P1).
- [ ] Confirm the V1-P9 `RichText.svelte` Tiptap extension registration pattern (how V1 registers marks/nodes) — the two node views follow it.

## File Structure

```
internal/content/blocks.go               node schemas + Set allowlist entries
internal/httpapi/block_resolve.go        batched resolution in the read walk
internal/richtext/render.go              figure placeholders (modify)
web/src/lib/components/RichText.svelte   node extensions + views (modify)
web/src/lib/components/BlockPickers.svelte  product/video pickers
web/e2e/v3p5-blocks.spec.ts              UAC-3.3
docs/architecture/04-api-layer.md        block resolution contract (amend)
docs/architecture/06-admin-ui.md         editor blocks note (amend)
```

---

### Task 1: Node schemas + write validation

**Files:**
- Create: `internal/content/blocks.go`
- Modify: `internal/content/document.go` (richText walk allowlist)
- Test: `internal/content/blocks_test.go`

**Interfaces:**
- Produces: `content.BlockProductType = "productBlock"`, `content.BlockVideoType = "videoBlock"`; walk rules: both are **leaf** nodes (a `content` key on them → 422 naming the node); `productBlock.attrs` = exactly `{collectionId, recordId}` (UUID strings); `videoBlock.attrs` = exactly `{collectionId, recordId, field}` (`field` a legal slug); unknown attr keys → 422; **no existence check at write** (UAC-3.3 survival semantics — the V1 relation shape-check philosophy).

- [ ] **Step 1: Failing tests** — table: valid product block saves inside a richText doc; valid video block saves; non-UUID `recordId` → 422 naming the node and attr; extra attr `price` → 422; block with children → 422; a block referencing a random (nonexistent) UUID **saves fine**.
- [ ] **Step 2:** FAIL. **Step 3:** implement (allowlist entries + one attr-shape validator each). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: productBlock/videoBlock canonical nodes with shape-only validation (F-31)"`

---

### Task 2: Public read resolution

**Files:**
- Create: `internal/httpapi/block_resolve.go`
- Modify: the public serialization walk (V2-P4 seam), `docs/architecture/04-api-layer.md`
- Test: `internal/httpapi/block_resolve_test.go`

**Interfaces:**
- Produces: on every public read (single, list, search), richText values are deep-walked for the two block types; attrs collected page-wide; **one batched query per block type**: products via the builder's anonymous-strict published-only form over each referenced collection (grouped by `collectionId`), hidden fields stripped per that collection's field rules for this audience; videos via the same record query reading just the named field. Each block node gains a sibling key: `"resolved": {"id", "collectionSlug", "data": {…visible fields…}, "seo": …}` for products; `"resolved": {"uid", "paywalled"}` for videos (**never a token**); missing/trashed/unpublished/invisible target, unknown collection, or non-video/absent `field` → `"resolved": null` — **the node and the document otherwise byte-identical** (UAC-3.3). Admin reads: raw canon, no `resolved` keys. Stored JSONB never mutated (resolution is response-only).

- [ ] **Step 1: Failing tests** — **`TestUAC_3_3_ProductBlockResolvesAndSurvivesDeletion`** (integration): publish product P and post X embedding a `productBlock(P)`; public read of X → the block node carries `resolved.data` with P's visible fields including the price field; **purge P → re-read X → 200, block present, `resolved: null`, every other node untouched** (deep-equal against the pre-deletion doc minus the resolved key); trash-then-restore P → resolved goes null then returns; hidden field on P absent from `resolved.data`; video block on a paywalled field resolves `{uid, paywalled:true}` with no token; two posts embedding five blocks over two collections → exactly 2 batched product queries (query-counter seam); admin read of X carries no `resolved`; `?format=json` byte-stability: stored canon unchanged after resolution reads.
- [ ] **Step 2:** FAIL. **Step 3:** implement + 04 amendment (block-resolution contract paragraph: posture, null rule, batching). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: batched anonymous-strict block resolution with null survival (UAC-3.3)"`

---

### Task 3: HTML placeholders

**Files:**
- Modify: `internal/richtext/render.go`
- Test: extend `internal/richtext/render_test.go`

**Interfaces:**
- Produces: renderer allowlist gains both nodes as attribute-only placeholders (spec §D5): `<figure data-cms-block="product" data-collection-id="…" data-record-id="…"></figure>` and `<figure data-cms-block="video" data-record-id="…" data-field="…"></figure>` — attr values HTML-escaped, no body text, renderer stays a pure function.

- [ ] **Step 1: Failing tests** — golden HTML for each block; attr escaping (a `recordId` of `"><script>` — impossible past Set validation but the renderer never trusts input — escapes inertly); blocks inside a blockquote render in place.
- [ ] **Step 2:** FAIL. **Step 3:** implement (two allowlist entries). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: block figure placeholders in HTML rendering (F-31)"`

---

### Task 4: Editor node views + e2e

**Files:**
- Create: `web/src/lib/components/BlockPickers.svelte`, `web/e2e/v3p5-blocks.spec.ts`
- Modify: `web/src/lib/components/RichText.svelte`, `docs/architecture/06-admin-ui.md`

- [ ] **Step 1: Failing e2e** — the UAC-3.3 arc in the browser: create+publish a product (commerce-enabled collection from the V3-P1 fixture); in a post's editor, toolbar "Insert product" opens the picker (lists commerce-enabled collections, then a record search), insert → card renders in the editor with the product's display text; save + publish; `page.request` public read → `resolved.data` present; delete the product via UI (trash → purge); public read → `resolved: null` and the post body otherwise intact; the editor re-opened shows the block card in a "missing reference" state (admin fetch 404 → card renders the warning, never crashes the editor).
- [ ] **Step 2: Implement** — Tiptap `Node.create` for each block (atom: true, no content); Svelte node views: product card (admin-fetches display data; 404 → missing-reference state), video card (record + field label, paywalled badge); `BlockPickers.svelte`: two-step picker (collection → record typeahead; video variant adds the field select filtered to video fields). Toolbar buttons appear only when eligible targets exist (any commerce-enabled collection / any collection with video fields). 06 amendment: one paragraph on editor blocks.
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: product/video block editor with missing-reference states (UAC-3.3)"`

---

### Task 5: V3 gate — programme acceptance sweep

- [ ] `make test` green; `make trace` green; `grep -c '^BR-' docs/trace-waivers.txt` → `0`.
- [ ] **UAC-3.1**: automated arc (V3-P3 stub suite) green + the **manual Stripe test-mode step** (V3-P3 T5 wording) executed and recorded.
- [ ] **UAC-3.2**: `TestUAC_3_2_PaywalledTokenArc` green + the **manual CF Stream playback step** (V3-P4 T5 wording) executed and recorded.
- [ ] **UAC-3.3**: `TestUAC_3_3_ProductBlockResolvesAndSurvivesDeletion` + `v3p5-blocks.spec.ts` green.
- [ ] V1 and V2 gates stay green: full e2e suite (V1 UAC + v2p1–p8 + v3p1–p5), `make bench` meets N-3/N-4.
- [ ] Mode matrix: boot with no V3 env (both 503 modes), Stripe-only, video-only, both — startup clean in all four, guards correct.
- [ ] Doc future-tense scan: `grep -n "(V3)" docs/architecture/*.md docs/BUSINESS_RULES.md` — every remaining hit is historical/scope text, not an unshipped promise; `11-roadmap.md` V3 gate row satisfiable as written.
- [ ] Programme close: seventeen + thirteen = **twenty-two plans** (9 V1 + 8 V2 + 5 V3) all executed; the roadmap's three releases are code.

## Self-Review Notes (execution-time attention)

- **`resolved` as a sibling key inside the node** (not a replacement of attrs) keeps the canonical doc recoverable from any response and makes the null case structurally identical to the found case — consumers branch on one key.
- **No new audit actions**: embedding a block is a record update (`content.record.update` already fires); resolution is a read. Adding block-level audit would double-count writes.
- **Editor missing-reference state** is the admin-side mirror of `resolved: null` — both sides of UAC-3.3's "without breaking" get an explicit test.
- **Toolbar gating on eligible targets** avoids dead buttons on content-only instances — commerce-less deployments never see product blocks; flagged as UX interpretation.
- **Shared allowlist source** (preamble item 1): if `content.Set` and `richtext` hold separate node lists by V3 time, unifying them here is in-scope hardening — divergent allowlists are how "validates but won't render" bugs are born.
