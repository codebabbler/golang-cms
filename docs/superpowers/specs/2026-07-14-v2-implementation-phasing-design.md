# V2 Implementation Phasing — Design

**Date:** 2026-07-14 · **Status:** Approved pending user review · **Scope:** Version 2 (Polish & Search) of golang-cms per `docs/architecture/11-roadmap.md`

## Context

V1 is fully planned (nine plans in `docs/superpowers/plans/v1/`) but not yet executed. V2 planning happens now, at the user's direction, producing execution plans only — no code until execution is initiated. Two facts shape this spec:

1. **The architecture docs are scope-grade, not implementation-grade, for V2.** `07-data-model.md` reserves the V2 table names (`cms_audit_log`, `cms_redirects`, `cms_webhooks`, `cms_webhook_deliveries`) and states their "layouts are specified in their delivery cycles." This spec is that delivery-cycle design record: §D pins every deferred layout at implementation grade, and each phase plan ships the matching architecture-doc amendment so the authority chain (`BUSINESS_RULES.md` > `architecture/*` > skills > code) holds by the time code lands.
2. **V2 plans are written against V1 plans, not V1 code.** The D-V1-3 re-validation rule binds doubly here: every V2 plan's execution opens with a re-validation pass against the then-current codebase, and the architecture docs win over any drifted plan detail.

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-V2-1 | Eight focused phases, groove-first (small self-contained features first, big rocks last) | Early phases are least sensitive to V1 drift and untag both V2-bound BRs quickly; execution rhythm is established before the risky work. Matches V1's granularity sweet spot. |
| D-V2-2 | Vertical slices: every phase ships backend + admin UI screens + Playwright e2e | By execution time V1-P9 has built the SPA shell, API client, stores, and router; screens are cheap increments, and each UAC-2.x needs UI anyway. Sole exception: V2-P4 is API-only (no admin-facing surface); its slice is verified by API-level e2e instead of a screen. |
| D-V2-3 | All eight plans authored up front in `docs/superpowers/plans/v2/`; each plan opens with the D-V1-3 re-validation preamble | Inherits the amended D-V1-3 discipline. Docs win over drifted plan detail. |
| D-V2-4 | V2 design decisions live in this spec at implementation grade; each phase plan ships the matching doc amendment (03/04/07/08, and `BUSINESS_RULES.md` where an invariant changes or is added) | One cycle instead of a separate docs-first pass; follows the V1 precedent (V1-P2's `cookie_set_at` + 07 amendment). |
| D-V2-5 | M2M relations (V2-P7) land before import/export (V2-P8) | Export serializes the final schema feature set — including `cj_` join tables — exactly once; no cross-phase retrofit task. |
| D-V2-6 | GDPR erasure (F-33) pairs with import/export in V2-P8 as a "data rights & compliance" phase | Thematic coherence (portability out + erasure); keeps V2-P2 small for the groove-first ramp. |
| D-V2-7 | Trace flip at V2-P1: `scripts/trace.sh` drops its V2-tag exemption (V3 exemption stays); the waiver file is seeded with exactly `BR-LIFE-9` and emptied by V2-P2 | BR-AUDIT-3 is tested inside V2-P1 itself, so it never enters the waiver. The V2 gate requires the waiver file empty — the same literal check as V1. |
| D-V2-8 | New BR-IDs ship with their phases: where a V2 feature carries an invariant with no BR-ID today, that phase's plan amends `BUSINESS_RULES.md` (rule + enforcement point) and lands the trace-satisfying tests in the same phase | Rule and test land together — no waiver entries. Planned additions: BR-HOOK-1/2 (V2-P6), BR-AUTH-15 (V2-P8), and the BR-SCHEMA-8 closed-set amendments (V2-P2, V2-P3). |
| D-V2-9 | Plan directories split by version: existing nine plans move to `docs/superpowers/plans/v1/`; V2 plans are written to `docs/superpowers/plans/v2/` (user direction). Docs-era plans (reviews, enablement) stay at the `plans/` root; specs stay flat in `specs/` | Separation of concerns; no file references plans by path except two generic mentions (fixed in this commit). |

## A. Phase Map (binding sequence)

All phases assume V1 complete (V1-P1…P9 executed, V1 gates green). Migrations continue the V1 ledger (V1 ends at 0002).

| Phase | Scope | Migration | UAC / BR anchors | Exit gate |
|---|---|---|---|---|
| **V2-P1 Audit log** | `cms_audit_log` sink behind the existing `audit.Recorder` interface (call sites untouched — BR-AUDIT-2); append-only (BR-AUDIT-3); admin audit browser filtering by actor, action, entity, time over the closed action vocabulary; **V2 harness flip** (trace.sh V2 exemption removed, waiver seeded with `BR-LIFE-9`) | 0003 | UAC-2.4 (audit half); BR-AUDIT-2/3 | Audit events from a schema change, a publish, and a key revocation visible in the UI; BR-AUDIT-3 traced; V1 gates green |
| **V2-P2 Scheduled publishing** | `publish_at` system column (BR-SCHEMA-8 amendment); `jobs.Publisher` 30 s ticker against DB `now()`; first-tick catch-up with late-publish logging; schedule/unschedule endpoints; admin datepicker + scheduled badge | 0004 | UAC-2.3; BR-LIFE-9; EC-13 | UAC-2.3 e2e; `BR-LIFE-9` leaves the waiver → **waiver file empty**; V1 gates green |
| **V2-P3 Redirects + SEO** | `cms_redirects` CRUD + public lookup endpoint (F-21, EC-4 rename bridge); `seo` system column (BR-SCHEMA-8 amendment) with closed key set, distinct exposure in responses (F-20); slug blocklist gains `seo`; redirects screen + SEO editor panel | 0005 | UAC-2.4 (redirect half); F-20/F-21 | Redirect lookup resolves with target and status code; SEO round-trips through save/publish/restore; V1 gates green |
| **V2-P4 Public API polish** | Public keyset cursors (F-27); public `contains` restored behind per-field `pg_trgm` GIN (BR-API-4); `?format=html` server-side Tiptap rendering (F-19) | 0006 (`pg_trgm` extension) | F-19/F-27; BR-API-4 | Cursor + offset coexist publicly; `contains` public only on trgm-indexed fields; HTML rendering canonical-JSONB-preserving; V1 gates green |
| **V2-P5 FTS** | `search_config` validation + editor UI; `cms_tiptap_text(jsonb)` immutable SQL extractor; generated `search_tsv` column + GIN per configured collection, regenerated inside the advisory-locked schema tx (EC-12); ranked search endpoints (public + admin); rebuild-duration audit events | 0007 (SQL function) | **UAC-2.1**; F-18; EC-12 | UAC-2.1 e2e (text + Tiptap plaintext, config change reindexes); V1 gates green |
| **V2-P6 Webhooks** | `cms_webhooks` registry + `cms_webhook_deliveries` outbox; HMAC-signed payloads; bounded in-process retry with post-commit nudge + fallback ticker; SSRF egress rules (BR-HOOK-1/2 added); purge-on-publish note + cache-TTL doc amendment (04); webhook management + deliveries viewer with manual redelivery | 0008 | UAC-2.2 (webhook half); F-22 | Publish delivers a signed webhook within 30 s e2e; SSRF suite green; V1 gates green |
| **V2-P7 M2M relations** | Schema engine cardinality `many`: `cj_` join tables with rename cascade (EC-4 extension); `Document.Set` + revisions snapshot/publish reconciliation; `ExpandRelations` over join tables; relation-editor UI | — (dynamic DDL) | F-23 | M2M create/edit/publish/restore round-trip e2e; V1 gates green |
| **V2-P8 Import/export + GDPR** | Schema export/import (first-class, covers final feature set incl. `cj_`); content export (NDJSON) first-class; content import best-effort with documented limitations; GDPR erasure (F-33, BR-AUTH-15 added): one-tx hard delete + token deletion + `created_by` tombstone rewrite; revision-redaction routine in `jobs.Retention` + admin endpoint; import/export screens + erase flow with typed confirm | — | UAC-2.2 (export half); F-25/F-33 | UAC-2.2 e2e both halves; **V2 gate**: UAC-2.1…2.4 pass, waiver empty, V1 gates green, bench unchanged |

**Dependencies beyond linear order:** V2-P8 depends on V2-P7 (schema features final before export). All other orderings are by size, not hard dependency — a stalled phase does not deadlock the programme.

## B. Cross-Phase Rules

1. **Re-validation preamble (inherited D-V1-3).** Every plan's execution opens with a re-validation pass against the then-current codebase; architecture docs win over drifted plan detail.
2. **Trace/waiver discipline.** V2-P1 removes the `(V2)` tag exemption from `scripts/trace.sh` (the `(V3)` exemption stays) and seeds `docs/trace-waivers.txt` with exactly `BR-LIFE-9`. V2-P2 empties it. New BRs (D-V2-8) are tested in-phase and never enter the waiver. The V2 gate check is literal: `grep -c '^BR-'` on the waiver file returns 0.
3. **Audit continuity.** Every new mutation surface (redirect CRUD, webhook CRUD, schedule/unschedule, search-config change, import, erasure, redaction) wires an audit call site (BR-AUDIT-1) with actions in the closed `domain.entity.verb` vocabulary; the vocabulary list in the audit package grows in the same commit as each new call site.
4. **No new env vars.** No V2 feature requires configuration beyond the V1 set. Test seams that would tempt an env var (e.g., allowing private-network webhook targets in e2e) are internal options on the component, never configuration.
5. **Migrations.** V2 continues the embedded ledger: 0003–0008 as mapped above. System-column additions to live tables (0004 `publish_at`, 0005 `seo`) use the backfill pattern: a SQL `DO` block iterates `cms_collections` and `ALTER TABLE`s each quoted `c_<slug>` table, and `createTableDDL` gains the column in the same phase so new tables match (BR-SCHEMA-8 stays a closed set, amended once per phase).
6. **Security invariants carry forward.** Never log tokens, cookie values, presigned URLs, JWT bodies — and now also webhook secrets and HMAC signatures. `TestNoSecretsInLogs` (V1-P8) gains webhook-secret regexes in V2-P6.
7. **Work on `main`; the user commits** unless commits are explicitly delegated. Done = phase acceptance sweep + `make test && make trace` green.

## C. Programme Acceptance Criteria

1. Eight plan files exist in `docs/superpowers/plans/v2/`, each opening with the re-validation preamble and closing with an acceptance sweep.
2. Every UAC-2.x maps to at least one phase exit gate (2.1→P5, 2.2→P6+P8, 2.3→P2, 2.4→P1+P3).
3. Waiver arithmetic closes: seeded {BR-LIFE-9} at V2-P1, empty from V2-P2 onward.
4. Each deferred layout in §D is owned by exactly one phase plan and one doc-amendment task.
5. No V2 plan reopens a V1 contract (roadmap sequencing rationale holds); where a V1 surface is extended (e.g., `createTableDDL`, `ExpandRelations`, `trace.sh`), the plan names the exact V1 interface it extends.
6. V1 gates remain green at every phase exit; the V2 gate additionally runs UAC-2.1…2.4 end-to-end.

## D. Design Record (implementation grade)

The subsections below pin every layout `07-data-model.md` deferred to "their delivery cycles," plus the cross-cutting shapes plans need. Each is normative for its phase plan; the phase ships the corresponding doc amendment.

### D1. `cms_audit_log` (V2-P1)

```sql
CREATE TABLE cms_audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    request_id  TEXT,
    actor_kind  TEXT        NOT NULL,   -- admin|api_key|end_user|anonymous|system
    actor_id    UUID,
    action      TEXT        NOT NULL,   -- closed domain.entity.verb vocabulary
    entity      TEXT        NOT NULL,   -- e.g. collection:posts/field:summary
    detail      JSONB,
    at          TIMESTAMPTZ NOT NULL
);
CREATE INDEX ix_cms_audit_log_at ON cms_audit_log (at DESC);
CREATE INDEX ix_cms_audit_log_actor ON cms_audit_log (actor_id, at DESC);
CREATE INDEX ix_cms_audit_log_action ON cms_audit_log (action, at DESC);
CREATE INDEX ix_cms_audit_log_entity ON cms_audit_log (entity text_pattern_ops, at DESC);
```

- **Dual sink.** The DB sink registers alongside the V1 `slog` sink behind the same `Recorder` interface; call sites are untouched (BR-AUDIT-2). The `slog` line always emits, so log-based alerting survives.
- **Failure degradation.** The DB insert runs synchronously with a 2 s timeout on the shared pool (not the caller's transaction). Insert failure logs `audit db write failed` at error and never fails the originating request — the `slog` line is the durable fallback.
- **Append-only (BR-AUDIT-3).** No UPDATE or DELETE code path exists; no API mutates or deletes rows; `jobs.Retention` gains no audit duty in V2 (growth is an operator concern, documented in the 08 amendment).
- **Admin API.** `GET /api/admin/audit-log` with filters `actor_id`, `actor_kind`, `action`, `entity` (prefix match), `from`/`to` (RFC 3339), keyset cursor on `(at, id)` descending. Read access: `admin`-and-above (viewer excluded — audit detail may reference entities the viewer cannot read).
- **System actor.** Machine-initiated events (scheduled publish, retention redaction) record `actor_kind='system'`, `actor_id=NULL`. Flagged for re-validation: if V1's `access.Principal` lacks a system kind, V2-P2 adds it.

### D2. Scheduled publishing (V2-P2)

- **Column.** `publish_at TIMESTAMPTZ NULL` joins the closed system-column set (BR-SCHEMA-8 amendment; 07 amendment). Backfill migration 0004 per rule B5. Partial index per table: `ix_<table>_publish_at ON (publish_at) WHERE publish_at IS NOT NULL`, added to `createTableDDL`.
- **Semantics.** Non-NULL `publish_at` on a non-trashed record means "publish the newest revision automatically at that instant" — this covers both never-published drafts and published records carrying a pending draft (the same two cases manual publish serves). Scheduling is a lifecycle operation: `POST /api/admin/collections/{slug}/records/{id}/schedule {publish_at}` (must be future — else 422; trashed record — 422), `DELETE …/schedule` clears it. Manual publish clears `publish_at`. Trash clears `publish_at`.
- **Publisher.** `jobs.Publisher` ticks every 30 s (08 already specifies both): one query per collection selecting `publish_at <= now()` (database `now()`, never process clock — 07) and, per hit, runs the same `lifecycle.Publish` path as a manual publish of the newest revision, as the system actor, then clears `publish_at`. Per-record failure logs at error and does not block other records; `publish_at` is retained so the next tick retries.
- **Catch-up (BR-LIFE-9, EC-13).** The first tick fires immediately after the listener opens (09 startup step) and is the catch-up scan: identical query, each late publication logged with its lateness. A 30 s ticker satisfies the 60 s bound.

### D3. Redirects + SEO (V2-P3)

```sql
CREATE TABLE cms_redirects (
    id          UUID PRIMARY KEY,                      -- UUIDv7, app-generated
    from_path   TEXT NOT NULL UNIQUE CHECK (from_path LIKE '/%'),
    to_path     TEXT NOT NULL,                         -- path or absolute URL
    status_code SMALLINT NOT NULL CHECK (status_code IN (301, 302)),
    created_at  TIMESTAMPTZ NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL
);
```

- **Lookup.** `GET /api/v1/redirects/lookup?path=/old-path` → `200 {data:{to_path,status_code}}` or `404`. Exact-match only in V2 (no wildcards/regex — YAGNI, documented). Anonymous responses follow the standard cache contract (BR-API-5). Admin CRUD at `/api/admin/redirects`; audit on every mutation.
- **SEO storage.** `seo JSONB NULL` joins the closed system-column set (BR-SCHEMA-8 + 07 amendments; backfill per B5). Closed key set, validated in `content.Document.Set`: `title`, `description`, `canonical_url`, `og_title`, `og_description`, `og_image` (media UUID, verified via the V1 `MediaVerifier`), `og_type`. Unknown keys → 422 naming the key.
- **SEO lifecycle.** `seo` rides the record pipeline: written through `Document.Set` under the reserved envelope key `seo` (the slug blocklist gains `seo`), snapshotted into `cms_revisions.data` under the reserved key `"$seo"` (illegal as a field slug, so no collision; drift mapping passes it through untouched), copied on publish/restore like content. API responses expose `seo` as a top-level sibling of `data` (F-20 "distinctly").

### D4. Public API polish (V2-P4)

- **Public keyset (F-27).** `/api/v1/collections/{slug}/records` gains the identical `?cursor=`/`meta.pagination.next_cursor` mechanism V1 built for admin lists — same cursor codec, same invalid-cursor 422, `cursor`+`offset` together → 422. Anonymous cursor pages are cacheable under the standard contract.
- **Public `contains` (BR-API-4 restoration).** Migration 0006 runs `CREATE EXTENSION IF NOT EXISTS pg_trgm` (bundled extension — N-9 holds). Field config gains `trgmIndexed bool` (schema engine: `CREATE INDEX … USING gin (col gin_trgm_ops)`, text fields only). `ScopePublic` accepts `contains` only on `trgmIndexed` fields; otherwise 422 naming the field, exactly as the V1 message shape. Admin/trash scopes unchanged. BUSINESS_RULES BR-API-4 text updated by this phase's amendment (rule already anticipates the restoration).
- **`?format=html` (F-19).** New `internal/richtext` package: a pure-Go Tiptap-JSONB→HTML renderer over a closed node/mark allowlist (doc, paragraph, heading 1–6, text, bold, italic, link with `rel="noopener nofollow"` and http(s)-only hrefs, bulletList, orderedList, listItem, blockquote, codeBlock, hardBreak, image over media URLs); unknown nodes render their text children only; all text HTML-escaped. Applies to `richText` fields on public reads when `?format=html`; stored JSONB remains canonical and untouched (04). ETag varies by format (the query string is part of the cache key upstream; the handler includes format in the ETag input).

### D5. FTS (V2-P5)

- **`search_config` shape** (column exists since migration 0001): `{"fields":[{"slug":"title","weight":"A"}]}` — weight ∈ A–D, field must exist with type `text` or `richText`, max 16 entries. Validated in the schema engine alongside `access_rules` validation; invalid config → 422 naming the offender.
- **Tiptap extraction.** Migration 0007 creates `cms_tiptap_text(doc jsonb) RETURNS text IMMUTABLE` — SQL implementation over `jsonb_path_query_array(doc, 'strict $.**.text')`, joined with spaces. (The Go-side walker for HTML rendering lives in `internal/richtext` from V2-P4; the SQL extractor is what a generated column can legally call.)
- **Materialization (EC-12, already normative in 03).** Setting a non-empty `search_config` adds, inside the same advisory-locked schema transaction: `search_tsv tsvector GENERATED ALWAYS AS (setweight(to_tsvector('english', coalesce(title,'')),'A') || …) STORED` plus `ix_<table>_search_tsv … USING gin (search_tsv)`. Any schema change touching a searched field regenerates column + index in the same transaction; clearing the config drops both. `'english'` is the fixed regconfig in V2 (documented; per-collection language is out of scope). The audit event records rebuild duration (08).
- **Search endpoints.** `GET /api/v1/collections/{slug}/search?q=…` and `GET /api/admin/collections/{slug}/search?q=…`: `websearch_to_tsquery('english', q)`, ranked `ts_rank_cd DESC, id ASC`, capped limit/offset only (no cursors — rank is not a stable keyset), empty/missing `q` → 422, unconfigured collection → 422 `search not configured`. Scope and visibility identical to list reads: the same `Decision` evaluation, publish floor, and field-rule filtering apply — rank never leaks invisible records or fields. Anonymous responses follow the standard cache contract.

### D6. Webhooks (V2-P6)

```sql
CREATE TABLE cms_webhooks (
    id            UUID PRIMARY KEY,               -- UUIDv7
    url           TEXT NOT NULL,                  -- http(s) only
    secret_sealed TEXT NOT NULL,                  -- AES-256-GCM, HKDF info="cms_webhook_secrets"
    events        TEXT[] NOT NULL,                -- ⊆ {record.create,record.update,record.publish,record.trash}
    collection_id UUID REFERENCES cms_collections(id) ON DELETE CASCADE,  -- NULL = all collections
    active        BOOLEAN NOT NULL DEFAULT true,
    created_at    TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL
);
CREATE TABLE cms_webhook_deliveries (
    id             UUID PRIMARY KEY,              -- UUIDv7
    webhook_id     UUID NOT NULL REFERENCES cms_webhooks(id) ON DELETE CASCADE,
    event          TEXT NOT NULL,
    payload        JSONB NOT NULL,
    status         TEXT NOT NULL CHECK (status IN ('pending','delivered','failed')),
    attempts       INTEGER NOT NULL DEFAULT 0,
    next_attempt_at TIMESTAMPTZ,
    last_response_code INTEGER,
    last_error     TEXT,
    created_at     TIMESTAMPTZ NOT NULL,
    delivered_at   TIMESTAMPTZ
);
CREATE INDEX ix_cms_webhook_deliveries_due ON cms_webhook_deliveries (next_attempt_at) WHERE status = 'pending';
```

- **Outbox pattern.** Lifecycle mutations enqueue delivery rows in the same transaction as the mutation (durable across restarts — 07's outbox note); a post-commit nudge channel wakes the deliverer, with a 15 s fallback ticker (satisfies UAC-2.2's 30 s bound even without the nudge). In-process only (BR-RUNTIME-5).
- **Signing (new BR-HOOK-1).** Secrets are generated server-side (32 random bytes, shown once at creation — same one-shot pattern as API keys), sealed at rest with the V1 keystore machinery under HKDF info `"cms_webhook_secrets"`. Each request carries `X-CMS-Delivery: <id>`, `X-CMS-Event`, `X-CMS-Timestamp` (RFC 3339), `X-CMS-Signature: sha256=<hex HMAC-SHA256(secret, timestamp + "." + body)>`. Secrets and signatures never appear in logs (`TestNoSecretsInLogs` extended).
- **Egress rules (new BR-HOOK-2, SSRF — 05 threat model).** At delivery time: resolve the host, reject private/link-local/loopback/multicast/unspecified addresses (v4 and v6), dial the resolved IP (pinned — no TOCTOU rebind), never follow redirects, 10 s per-attempt timeout, response bodies read at most 4 KiB and discarded. e2e tests use an internal allow-private option on the deliverer — never an env var (rule B4).
- **Retry.** 2xx = delivered. Otherwise up to 5 attempts total: the initial post-commit attempt plus four retries delayed 15 s, 2 m, 15 m, 1 h after each successive failure; after the 5th failure `status='failed'` (terminal). Admin can redeliver manually (resets to pending, attempts 0). `jobs.Retention` gains one duty: delete `delivered`/`failed` rows older than 7 days. Payload: `{delivery_id, event, occurred_at, collection:{id,slug}, record_id, actor_kind}` — IDs only, no record content, no PII.
- **Purge-on-publish.** `record.publish` is the purge signal; the 04 amendment documents that consumers may lengthen cache TTLs once subscribed. No new event type.
- **Telemetry.** `jobs.Webhooks` joins the job-telemetry section (08 amendment): per-tick delivered/failed/retried counts.

### D7. M2M relations (V2-P7)

- **Field model.** Relation field config gains `cardinality: "one" | "many"` (default `one` — existing V1 relations unchanged). `many` provisions no live-table column; it provisions a join table.
- **Join-table anatomy** (07 amendment; `cj_` prefix already reserved):

```sql
CREATE TABLE cj_<collection>_<field> (
    source_id  UUID NOT NULL REFERENCES c_<src>(id) ON DELETE CASCADE,
    target_id  UUID NOT NULL REFERENCES c_<tgt>(id) ON DELETE RESTRICT,
    position   INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (source_id, target_id)
);
CREATE INDEX ix_cj_<collection>_<field>_target ON cj_<collection>_<field> (target_id);
```

  Named by source collection slug + field slug with the V1 63-byte truncation rule (20-char + 8-hex-hash). `ON DELETE CASCADE` from source (purging a record purges its memberships); `ON DELETE RESTRICT` toward targets (purging a referenced target fails with `ErrReferenced`, matching V1 relation semantics).
- **Rename cascade (EC-4 extension, 03 amendment).** Renaming the source collection or the field renames the join table and its indexes in the same advisory-locked transaction; renaming the *target* collection touches nothing (FKs bind to the physical table by OID).
- **Write path.** `Document.Set` accepts an array of target UUIDs (max 500 per field per record — 422 beyond; duplicates rejected), verifies each target exists and is not trashed. Save writes the live row and reconciles the join table (delete-then-insert diff, `position` from array order) in the existing single write transaction; the revision snapshot stores the UUID array under the field slug. Publish/restore reconcile join rows to the chosen revision's array. Trash leaves memberships intact (restore is lossless).
- **Read path.** `ExpandRelations` covers `many` fields with one join-table query per field, targets filtered by the requester's visibility exactly as V1 (invisible targets are omitted from arrays, not nulled — array shape, unlike V1's scalar null). Filtering/sorting on M2M fields is rejected 422 in all scopes in V2 (YAGNI, documented).
- **FTS/export interplay.** M2M fields are never searchable (not text). Export (V2-P8) serializes memberships as UUID arrays.

### D8. Import/export + GDPR erasure (V2-P8)

- **Schema export.** `GET /api/admin/export/schema` → `{"format_version":1,"exported_at":…,"collections":[{slug,name,access_rules,search_config,fields:[{slug,type,config,position}]}]}`. No database IDs; relation targets referenced by slug.
- **Schema import.** `POST /api/admin/import/schema` (super_admin, recent-auth): validates the whole document first, then creates collections through `schema.Engine` in dependency order (relations after targets; cycles → two passes: tables then relation fields). Any existing collection slug → 409 naming it, nothing applied (strict fresh-instance semantics — UAC-2.2's scenario; merge is a documented V2 limitation).
- **Content export.** `GET /api/admin/export/content/{slug}` → NDJSON stream, one live row per line: `{id,status,seo,publish_at,created_at,updated_at,created_by,data:{…field slugs…, m2m fields as UUID arrays}}`. Live rows only (revisions are not portable — documented).
- **Content import.** `POST /api/admin/import/content/{slug}` (NDJSON body, best-effort, create-only): rows pass through `content.Document.Set` (BR-RBAC-5 — no bypass) and are created as drafts; a row whose export said `status='published'` is then published through the normal lifecycle path, preserving the BR-LIFE invariants. ID collision → row skipped; relation/media UUIDs kept only if the referenced row exists locally, else the field is nulled and reported; `created_by`/`created_at` are not preserved (the importer becomes the author). Response: `{imported, skipped:[{id,reason}]}`. Documented limitations per F-25 plus the above: ID collisions, relation remapping, media references (objects are not copied), authorship/timestamps not preserved.
- **Erasure (F-33, new BR-AUTH-15).** `POST /api/admin/end-users/{id}/erase` (super_admin, recent-auth, typed confirm = the user's email). One transaction: delete the `cms_end_users` row, all `cms_refresh_tokens` for the user, all `cms_reset_tokens` with `user_kind='end_user'` and that `user_id`; rewrite `created_by` to the tombstone UUID `00000000-0000-4000-8000-000000000000` across every `c_<slug>` table and `cms_revisions`. The audit event records the erased user's ID only (no email — the identity link is what's being erased). Admin UI resolves the tombstone as "Deleted user".
- **Revision redaction.** `jobs.Retention` gains a reusable `RedactRevisions(collection_id, record_id, fields []slug)` routine that strips the named keys from all of that record's `cms_revisions.data` snapshots; exposed at `POST /api/admin/revisions/redact` (super_admin, recent-auth). Limitations documented per 07: redacts known fields, not free-text mentions.

## E. Out of Scope (V2)

Everything in REQUIREMENTS §5 (O-1…O-12); V3 scope (carts, Stripe, paywalls, custom Tiptap blocks); redirect wildcards; per-collection FTS language; M2M filtering/sorting; schema-import merge mode; audit-log retention duties; webhook event types beyond the four lifecycle events.
