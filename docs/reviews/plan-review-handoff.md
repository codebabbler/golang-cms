# Plan Review Handoff — V1–V3 Inline Review

**Reviewer:** inline (no subagents, per user direction) · **Started:** 2026-07-14 · **Status:** V1 in progress

Purpose: working state for the 22-plan review so context compaction never loses findings or cross-version ledger. Findings accumulate per plan as reviewed; each version closes with a ledger section the next version's review consumes. Final consolidated report goes to `docs/reviews/2026-07-14-plan-review-v1-v3.md` for user triage (approved flow: report → user triage → fixes → re-review of changed docs).

## Rubric (approved design)

Per plan: (1) spec fidelity, (2) doc conformance (authority chain: BUSINESS_RULES > architecture/* > skills > plans), (3) internal coherence, (4) edge cases, (5) project-skill invariants. Cross-cutting per version + final: interface continuity, harness/waiver/migration arithmetic, BR+UAC coverage, EC-1..16 register, security lens, ops lens. NOT findings: "code doesn't exist", execution-time re-validation needs (designed preamble), style, Self-Review-Notes-flagged decisions unless they violate higher authority.

Severity: HIGH = wrong implementation/contract violation/gate failure would result. MEDIUM = implementer could plausibly get it wrong. LOW = worth noting.

## Review status board

| Plan | Status | Findings |
|---|---|---|
| v1p1 foundation (2905 ln) | pending | |
| v1p2 auth-core (611) | pending | |
| v1p3 schema-engine (191) | pending | |
| v1p4 content-core (228) | pending | |
| v1p5 principals (191) | pending | |
| v1p6 public-api (176) | pending | |
| v1p7 media (145) | pending | |
| v1p8 jobs-bench (128) | pending | |
| v1p9 admin-ui (158) | pending | |
| V1 cross-cutting pass | pending | |
| v2p1..v2p8 | pending | |
| V2 cross-cutting pass | pending | |
| v3p1..v3p5 | pending | |
| V3 cross-cutting + programme pass | pending | |

## Findings

(appended per plan as reviewed; ID format F-<plan>-<n>)

### v1p1 foundation — reviewed in full (2905 ln)

- **F-v1p1-1 (HIGH, environment-fit)** — Task 3 Step 5 + Task 9: `scripts/testdb.sh` hardcodes `docker`. The target dev machine's docker socket is permission-denied; podman 5.8.4 works rootless (documented planning-session constraint: container scripts must be runtime-agnostic). `make test` is unrunnable as planned. **Fix:** `engine="${CONTAINER_ENGINE:-$(command -v docker || command -v podman)}"` + use `$engine` throughout; same fix applies to v1p7's `scripts/testenv.sh` (verify when reviewing v1p7).
- **F-v1p1-2 (MEDIUM, cross-version)** — Task 5 `statusWriter` (Logger middleware) implements only WriteHeader/Write: no `Flush()`, no `Unwrap()`. Any downstream `http.Flusher` assertion fails through the middleware chain — v2p8 Task 2's NDJSON export streaming (`http.Flusher` flush per batch) silently stops streaming. **Fix:** add `Unwrap() http.ResponseWriter` (ResponseController-compatible) and a `Flush()` passthrough.
- **F-v1p1-3 (HIGH, doc conflict inherited)** — Task 3 DDL: `cms_api_keys.created_by UUID NOT NULL REFERENCES cms_users(id)` faithfully mirrors 07's row ("created_by FK"), but 07's admin-deletion contract (§created_by paragraph: "Deleting a `cms_users` row cascades its `cms_sessions` and `cms_reset_tokens` rows…") is unimplementable with it: revoked keys are retained forever (BR-AUTH-7), so any admin who ever created an API key can never be deleted (FK NO ACTION). 07 is self-contradictory; the plan inherited the conflict. **Fix:** drop the FK (created_by is the FK-less convention everywhere else — 07 even lists `cms_api_keys` as a created_by *source* store) + one-line 07 amendment; alternatively ON DELETE SET NULL + nullable column. Decision needed at triage.
- **F-v1p1-4 (HIGH, cross-version harness)** — Task 9 `trace.sh` waiver match is `grep -qE "^${id}[[:space:]]"` — requires a trailing annotation field. v2p1 Task 6 reseeds the waiver file with "exactly one entry: `BR-LIFE-9`" (bare). A bare line never matches → BR-LIFE-9 counts unwaived right after the V2 flip binds it → `make trace` FAILS at v2p1's gate. **Fix:** regex `^${id}([[:space:]]|$)` in P1's script, AND/OR v2p1 writes the annotated form `BR-LIFE-9 V2-P2` (file's stated format is `<BR-ID> <owning phase>` anyway). Cross-filed as F-v2p1-1.
- **F-v1p1-5 (LOW)** — SPA `index.html` served `no-cache` with no ETag/Last-Modified: revalidation is always a full re-download. Optional ETag would honor EC-15 cheaper.
- **F-v1p1-6 (LOW)** — No secondary indexes on `cms_sessions.user_id`, `cms_refresh_tokens(end_user_id)`/`(family_id)`, `cms_reset_tokens(user_kind,user_id)`: cascades, family revocation, and erasure scans seq-scan. Harmless at design scale; later phases may want them.

Verified-good (v1p1): waiver arithmetic exact (48 waived + 8 P1-tested + 4 structural + 2 V2-tagged = 62 BR-IDs, matches BUSINESS_RULES grep); 17-env-var set complete; nine-code registry + statuses; middleware order matches 04 (`RequestID → Logger → Recover` prefix); 13-table DDL spot-checked vs 07 (sessions/api_keys/reset_tokens/revisions partial-unique all match); advisory keys 0x636D7300/02 correct, 0x636D7301 reserved for P3; 07 version bump 1.3→1.4 correct; migration runner Sscanf/ledger/idempotency sound; boot/second-instance/watchdog/drain tests sound (loopback dial, timeout guards); `tested()` word-boundary regex sound incl. `\$` quoting.

### v1p2 auth-core — reviewed in full (611 ln)

- **F-v1p2-1 (MEDIUM)** — Task 6: `GET /api/admin/auth/csrf` rotates the CSRF value on every call (forced by hash-only storage). Two admin tabs: a refresh in tab A invalidates tab B's token → B's next mutation 403s with no recovery contract. **Fix:** v1p9's api client must auto-refetch CSRF once on `forbidden` and retry (one-line contract addition); alternatively rotate only on login. Cross-filed to v1p9.
- **F-v1p2-2 (LOW)** — Task 7: `RouterConfig` gains `SetupUserID func()` — garbled/purposeless (setup creates the first admin; no target-user id exists). Remove; only recovery needs `RecoveryUserID`.
- **F-v1p2-3 (LOW)** — Task 3: `Verify` touches `last_seen_at` on every authenticated request (one UPDATE per request). Correct but chatty; a >1 min threshold touch would halve session-table writes. Not a contract issue.

Verified-good (v1p2): Argon2id params/PHC/semaphore exact per BR-AUTH-3; DummyHash uniform timing; cookie attribute set exact; EC-10 table incl. spoofed-prefix case; LRU cap + eviction-forgets; migration 0002 ADD-with-DEFAULT-then-DROP pattern; waiver shrink 9 exact; step-5 always-warn token logger honors the hard-rule-6 exception; session fixation reasoning sound.

### v1p3 schema-engine — reviewed in full (191 ln)

- **F-v1p3-1 (HIGH, inherited doc contradiction)** — Task 5 vs 03-dynamic-schema.md: the plan (like 03 §Concurrency line 57) uses `pg_advisory_xact_lock`, which releases AT commit — but 03 line 88 and BR-RUNTIME-7 require the cache swap to happen "after commit, **before** releasing the advisory lock". With an xact-scoped lock that ordering is unsatisfiable, and two overlapping Applies can swap snapshots out of order (stale snapshot wins). **Fix:** serialize `Apply` with an in-process mutex (sound under single-instance BR-RUNTIME-8) and keep the xact lock for cross-process safety, or switch to session-scoped `pg_advisory_lock` released after the swap; amend 03 to name the mechanism. Triage decision needed (doc + plan touch).
- **F-v1p3-2 (LOW)** — Global Constraints say "exactly the nine operations" while Task 5 defines 11 op structs (03's table pairs Add/Drop rows). Add one clarifying sentence to prevent an implementer "fixing" the count.
- **F-v1p3-3 (LOW, observation)** — Relation fields carry no FK until an explicit `AddForeignKey` (03-sanctioned; dangling UUIDs render as null via BR-LIFE-6 expansion). Ensure v1p4/v2p7/v3 text never claims unconditional FK-enforced existence for relations (v1p4 Task 3's wording is fine; v2p7/v3p2 say "FK enforces existence" — soften at fix time).

Verified-good (v1p3): slug regex/blocklist vs BR-SCHEMA-2; caps 200/500 with stubbed-cap test rationale; conversion matrix cases match 03 verbatim; index truncation rule (20+8hex); DropCollection revision deletion (EC-2 asymmetry); rename EC-4 semantics; error→status mapping complete; waiver shrink 8 exact; SET LOCAL lock_timeout applies to table locks (03 line 65) — advisory-wait-unbounded is by design, not a bug.

### v1p4 content-core — reviewed in full (228 ln)

- **F-v1p4-1 (HIGH)** — Task 4: the revision snapshot merges "live values overlaid with doc values". For a **published record with a pending draft**, the live row holds the *published* content, so a second partial update merges against published values and silently discards the first draft edit (title→X in rev N+1, then body→Y produces rev N+2 = {title: published-A, body: Y}). **Fix:** merge base = the newest revision's data when `max(version_no) > published version_no` (i.e., the pending draft), else the live row. One sentence in Task 4 + one regression test (partial-update-twice-on-published).
- **F-v1p4-2 (MEDIUM)** — Task 6: the 5 MiB record body-cap class is applied only to `POST /` (create). `PUT /{id}` (update) — and revision-restore bodies — stay under the 64 KiB default, so any legitimate large richText **edit** 413s. Fix: the 5 MiB class covers the record-write route class (create + update), per 04's "5 MiB records".
- **F-v1p4-3 (MEDIUM)** — Task 3 states two conflicting `content.Set` signatures in one interface block (line 143 `Set(snap, col, rules, input) (Document, []FieldError)` vs the corrected `Set(ctx, snap, col, rules, input, mv MediaVerifier)`) and leaves the deliberation text in place. Keep only the final signature (the one v2/v3 plans consume).
- **F-v1p4-4 (LOW)** — Task 6 interim floor `RequireRole(editor)` on ALL mutations blocks the contributor persona entirely until P6 (contributors legitimately create drafts per 12 §1). Deliberate and greppable (`// P6:`), but note it in the plan text as a known persona gap, or floor creates at contributor.

Verified-good (v1p4): all seven builder invariants present with SQL-shape tests; owner-draft OR-form; cursor semantics + tiebreaker; trash/restore EC-6 collision mapping via index name (composes with the truncation helper); drift four rules; publish atomicity via partial unique index; BuildInsert/BuildUpdate flagged additive; waiver shrink 9 exact; unpublish flag semantics deliberate; purge/23503 coherent with optional relation FKs.

### v1p5 principals — reviewed in full (191 ln)

- **F-v1p5-1 (HIGH, doc conformance)** — Task 4 register: duplicate email → `422 "registration failed"` while success → `201 {accessToken, refreshToken}`. 05 §5 line 73 is explicit: "`login` and `register` return uniform errors … so neither response shape nor timing discloses account existence" — the 201/422 status split IS a shape oracle. **Fix (docs win):** register always returns the same success shape (`201 {data:{}}`, no tokens — auto-login dropped) after one Argon2id hash, duplicate silently no-ops; client logs in afterwards (login is already uniform). Touches: v1p5 Task 4 + Self-Review note, v1p9 client flow, openapi at execution.
- **F-v1p5-2 (MEDIUM)** — Task 5: `APIKeyService.Create(ctx, name, scopes)` omits the actor — `cms_api_keys.created_by` is NOT NULL. Add `createdBy uuid.UUID` (or take the Principal).
- **F-v1p5-3 (MEDIUM)** — `/api/admin/users` ships list/create/reset-token only. The file-structure block promises "role" management, and 07's admin-deletion paragraph ("Deleting a `cms_users` row cascades its sessions and reset tokens…") contracts deletion — neither a role-change route nor `DELETE /{id}` exists in any plan. Combined with F-v1p1-3 (api-keys FK), admin lifecycle management is under-planned. **Fix:** add `PUT /{id}/role` (super_admin floor, last-super_admin guard) and `DELETE /{id}` (RequireRecentAuth + typed confirm; app-level reset-token cascade per P1's deviation note) to v1p5 Task 6, or explicitly document admins-are-never-deleted and amend 07 — triage decision.
- **F-v1p5-4 (LOW)** — Deliberation text left inline (Task 4 register "…`201 {data:{}}`? No —"; same class as v1p4's Set-signature waffle). Strip to final decisions only.
- **F-v1p5-5 (LOW, design note)** — Live refresh tokens never expire (no `expires_at`; only rotated/revoked rows are purged by retention). 05 is silent — an unused refresh token works forever. Suggest a docs-level decision (e.g., 90-day absolute family lifetime) rather than a plan patch.

Verified-good (v1p5): keystore sealing/HKDF info string + fail-closed load; JWT claims-exactly + alg-confusion + kid rotation-overlap tests; refresh rotate/reuse-family-revocation exactly EC-8; registration gate 404; revoked-key→anonymous with nil error; cookie-ignored-on-/api/v1; reset trusted-caller 404 exception matches 05 §3 verbatim; disabled→anonymous per-request DB check mandated by BR-AUTH-8; waiver shrink 6 exact.

### v1p6 public-api — reviewed in full (176 ln)

- **F-v1p6-1 (LOW)** — Task 3: the `count=exact` anonymous gate lives "in ParsePagination", but P4's `ParsePagination(r)` has no principal parameter — name the signature growth explicitly (`ParsePagination(r, p)` or ctx-principal) so P4/P6 don't ship diverging helpers.

Verified-good (v1p6): evaluator implements 12 §3 order with the three worked examples as normative tests; eight §4 rejections; CORS preflight-before-auth with saturation-exemption test; ETag/304/Vary exact BR-API-5; idempotency race protocol (poll→replay / 409-retry) consistent with 04; expansion anonymous-strict + query-count assertion; createStatus one-tx published create; interim-floor replacement task greps `// P6:`; waiver shrink 11 exact; post-Auth anonymous bucket flagged as documented interpretation.

### v1p7 media — reviewed in full (145 ln)

- **F-v1p7-1 (MEDIUM)** — Task 4 floors presign/finalize at `editor+`, locking contributors out of uploads entirely — a contributor drafting a post with a Media widget gets 403 on presign, permanently (this floor is never replaced by Decisions). Fix: floor presign/finalize at `contributor+` (upload grants nothing content-wise; attachment is still Decision-gated) — or route them through Decide(create) on some collection.
- **F-v1p7-2 (LOW)** — Task 2 says media-less mode makes "every method" return `ErrUnavailable", but the Global Constraints require listing to keep working. True resolution (list rides sqlc directly) is only implicit — reword to "every storage-touching method".
- Corroborates **F-v1p1-1**: Task 1 says testenv.sh uses "the same runtime-detection pattern as testdb.sh (docker, else podman)" — v1p1's testdb.sh has no such pattern. The detection intent exists; v1p1's script dropped it.

Verified-good (v1p7): presign/HEAD/delete contract with signed Content-Type/Length; 404-success deletes; one-tx row+outbox with crash-safety test; ErrReferenced table+record naming; MediaVerifier goes live with pending-reference 422; media-less 503 + always-on listing; object-key sanitization; waiver shrink 3 with the BR-MEDIA-2 two-halves note.

### v1p8 jobs-bench — reviewed in full (128 ln)

- **F-v1p8-1 (HIGH, doc conformance)** — BR-AUDIT-1 explicitly enumerates **purge** among audited mutations; retention duty 1 hard-purges trash with NO audit emission, and V1 has no system actor at all (`KindSystem` only arrives in v2p2). Fix: pull `access.KindSystem` + `SystemPrincipal()` forward into v1p8 (or P1), duty 1 emits `content.record.purge` audit events per purge; v2p2 Task 2 becomes a consumer, not creator. (Expiry purges of sessions/tokens are not in BR-AUDIT-1's enumeration — no audit needed there.)
- **F-v1p8-2 (MEDIUM)** — Task 5 bench fires ~600 anonymous public reads inside a minute; BR-API-7's 300/min-per-IP limiter (live since P6) 429s the bench halfway. Fix: bench requests authenticate (API key principal — no anonymous bucket, still full-stack) or the bench must note/handle the limiter; do not disable the limiter via config (none exists — by design).
- **F-v1p8-3 (LOW)** — Task 3's pruning SQL sketch carries a vestigial `first_value(...)` window that the final predicate never uses; drop it from the sketch to avoid faithful transcription of dead SQL.

Verified-good (v1p8): duty order matches 07 §Retention verbatim; FK-skip warn line shape is the 08 alert input; pruning exclusion test pins newest+published; object-first ordering both media duties; EC-14 stop-tickers-before-drain wiring; log-assert regex set matches the never-log list with exactly the two carve-outs; CopyFrom bypass justified+tagged; waiver empties with the P1-exemption arithmetic.

### v1p9 admin-ui — reviewed in full (158 ln)

- **F-v1p9-1 (MEDIUM, completes F-v1p2-1)** — the api client contract handles 403+`recent_auth_required` but has no recovery for CSRF-rotation 403s (multi-tab). Fix pair: `RequireCSRF` returns `details:{"reason":"csrf_mismatch"}`; client refetches CSRF once and retries on that reason.
- **F-v1p9-2 (raises F-v1p5-3 to confirmed)** — Task 9 ships role-management UI ("role management within persona limits"), but NO plan defines a role-update endpoint. v1p5 must add `PUT /api/admin/users/{id}/role` (super_admin-only for super targets, last-super_admin guard).
- **F-v1p9-3 (LOW)** — Task 2's CSP execution-check covers Vite inline module preloads but not `style-src` — ProseMirror/Tiptap injects inline styles in places; add style-src to the same execution check with a named remediation.

Verified-good (v1p9): client contract (envelope, typed errors, CSRF-in-memory, 401 redirect, RecentAuthRequired funnel); CSP exact per 06 with media-origin handling; client-side mirrors tested against normative fixtures; typed-confirm + pre-flight re-auth on every destructive flow; Tiptap canonical JSONB round-trip test; Stage A/B split sanctioned; pointer-into-06 style justified (06 is normative + thrice reviewed).

### V1 cross-cutting pass — complete

- **F-v1x-1 (MEDIUM, doc + plans)** — REQUIREMENTS §6 defines **UAC-1.1…1.7**, but `11-roadmap.md`'s V1 gate row, the V1 phasing spec, v1p9's goal/Task-10 text, and every V2/V3 "V1 gates stay green" sweep say "UAC-1.1…1.6". §6 governs (the roadmap defers to it explicitly). UAC-1.7's *content* is covered (v1p5 disable arc, v1p6 createStatus/owner-draft tests), so this is an enumeration fix: roadmap gate row, v1p9 (2 places), and the sweep wording. v1p9 Task 10 already hedges ("re-read REQUIREMENTS at execution") — make it unconditional.
- **Interface continuity (V1):** verified while reading — CountUsers P1→P2; RequireSession/CSRF/RecentAuth/Role P2→P3+; Snapshot/Cache/CanConvert P3→P4; ParseFilters/ParsePagination + MediaVerifier + EncodeCursor P4→P6/P7/V2; ListMediaDeletionsOlderThan P7→P8; pendingDraft + recent_auth detail shape + `make e2e` P4/P2/P9→P9/V2/V3. One drift: F-v1p6-1 (ParsePagination principal). No dangling consumes found.
- **Harness arithmetic:** waiver 48 = 62 BRs − 8 P1-tested − 4 structural − 2 V2-tagged, per-phase shrinks 9/8/9/6/11/3/2 sum exactly; migrations 0001–0002; trace.sh sound except F-v1p1-4 (bare-entry regex).
- **EC register:** EC-1…16 each land in a named V1/V2 task (EC-12→v2p5, EC-13→v2p2; all others V1). No orphans.
- **Security lens:** authz floors present on every new route; never-log list enforced by P8 sweep; F-v1p5-1 (register oracle) is the one conformance break; F-v1p7-1 is an authz-granularity gap; no missing CSRF/recent-auth on destructive flows found.
- **Ops lens:** startup steps accrue in order (P1 seams → P2 step5 → P3 step4 → P5 keystore → P8 scheduler); EC-14 ordering correct once P8 lands; F-v1p1-2 (Flusher) is the one plumbing gap.

### v2p1 audit-log — reviewed in full

- **F-v2p1-1 (HIGH, carry-in confirmed; self-detecting)** — Task 6 Step 2 reseeds the waiver with the bare line `BR-LIFE-9`; trace.sh's match regex `^${id}[[:space:]]` requires a trailing field, so the entry is invisible → `make trace` goes RED at Step 3's own verification. Plan ships known-red instructions even though the executor would spot it. Fix jointly with F-v1p1-4: regex `^${id}([[:space:]]|$)` in v1p1 AND write the annotated form `BR-LIFE-9 V2-P2` here.
- **F-v2p1-2 (LOW)** — Self-Review note "system actor intentionally absent until V2-P2" becomes false once F-v1p8-1 pulls `KindSystem` into V1 (retention purge audits); update the note and ensure the audit browser's actor-kind filter includes `system`.

Verified-good (v2p1): §D1 DDL verbatim; sqlc narg/row-comparison keyset SQL valid; reflection-based no-mutation test; dual-sink degradation with detached 2s ctx; viewer exclusion; flip mechanics + self-verifying trace step; handler-local bigint cursor decision sound.

### v2p2 scheduled-publishing — reviewed in full

- **F-v2p2-1 (MEDIUM, carry-in confirmed)** — migration 0004's DO block builds `'ix_c_'||slug||'_publish_at'` raw; V1's slug cap is 55 chars (`{0,54}`), so the name can reach 71 bytes and Postgres silently truncates to 63 — diverging from `indexName()`'s 20+8-hash rule, breaking rename-cascade and IF-NOT-EXISTS dedup for long slugs. Fix: the DO block must implement the same truncation rule in plpgsql (substr + sha256 via pgcrypto/encode — or precompute names in a preceding SQL SELECT), plus a parity test comparing plpgsql and Go naming for a >63-byte case. Same audit applies to 0005's seo backfill (no index — clean).
- **F-v2p2-2 (LOW)** — `query.SetPublishAt` writes `updated_at = now()` (DB clock) while the V1 convention is app-side UTC timestamps; align or document.
- **F-v2p2-3 (LOW)** — Preamble cites "V1-P3 T?" (it's T5); Unschedule's role floor is implied, not stated — make both explicit.

Verified-good (v2p2): BR-SCHEMA-8 amendment + anatomy-test growth; DB-now() semantics incl. ErrScheduleInPast via DB clock; drain-loop catch-up satisfying EC-13 with LIMIT rationale; per-record failure isolation retaining publish_at; system-principal lattice ride; publish/trash clearing; waiver emptied with stale-check leverage; pending-draft-of-published scheduling included (spec self-review fix carried through).

### v2p3 redirects-seo — reviewed in full

- **F-v2p3-1 (MEDIUM)** — `seo` rides every record write with no principal gating: end_user/api_key writers (public creates, createStatus flows) can set `canonical_url`/OG fields on records that may publish immediately — SEO spam/abuse surface. Neither spec §D3 nor 12's field rules cover the reserved envelope key. Fix: reject `seo` in the request body for non-admin-kind principals (422 naming `seo`) unless a future grant opts in; document in 04's write-shape amendment.
- Ledger note: the `$seo` drift-rule-0 passthrough interacts with F-v1p4-1's merge-base fix — when the merge base becomes the pending revision, `$seo` must merge from the same base (one fix site).

Verified-good (v2p3): freeze semantics literal; `$seo` reserved-key collision-free with slug grammar; blocklist growth; redirect DDL/CHECKs; lookup cache contract + 304; loop-detection stance documented; audit vocabulary domain growth forced by the V2-P1 parser test.

### v2p4 public-api-polish — reviewed in full

- **F-v2p4-1 (LOW)** — Renderer mark allowlist (bold/italic/link) is narrower than the starter-kit marks P9's editor emits (`code`, `strike`): those marks silently drop in HTML output. Either add `code`→`<code>`/`strike`→`<s>` or configure the P9 editor to exclude them — pick one side and test it.
- **F-v2p4-2 (LOW)** — Task 3's XSS corpus should add case/whitespace scheme variants (`JaVaScRiPt:`, `\tjavascript:`) and pin that href/src validation goes through `url.Parse` scheme comparison, not string prefix.

Verified-good (v2p4): trgm index DDL in the whitelist with type gating + rename carry; contains gate condition exact; cursor exposure reuses V1 mechanics with the old 422 case deleted-not-duplicated; renderer contract (leaf escaping, unknown-node text-only, depth cap, pure function); format=html canonical-untouched byte test; ETag-by-body making format variants safe; ILIKE-confinement grep.

### v2p5 fts — reviewed in full

- **F-v2p5-1 (MEDIUM)** — The EC-12 regen hook covers field **drop** and **type-change** but not **rename**: `search_config` stores field slugs, so renaming a searched field leaves the config naming a dead slug (validation of the next op fails; the drop-hook's slug match misses). Fix: RenameField, when the field is in `search_config`, rewrites the config slug in the same tx (the generated column itself survives renames — PG tracks by attnum).
- **F-v2p5-2 (MEDIUM)** — tsvector values cap at ~1 MiB; richText fields allow 1 MiB each and up to 16 searched fields, so the generated expression can exceed the limit — making **row writes fail on searched collections** with large content. Fix: wrap the extractor in a length cap (`left(cms_tiptap_text(…), 262144)` or similar) inside the generated expression and document the indexing truncation in 03's amendment.
- **F-v2p5-3 (LOW, self-detecting)** — `jsonb_path_query` (set-returning, in a subquery) inside a `LANGUAGE sql` function backing a GENERATED column is *probably* legal on PG16 (the function isn't inlined because of `string_agg`), but it's the plan's shakiest Postgres assumption; Task 2's first test fails fast if wrong. Name the fallback in the plan: LANGUAGE plpgsql equivalent, or built-in `to_tsvector(regconfig, jsonb)` accepting node-type-name noise.

Verified-good (v2p5): config shape/validation; regen-in-triggering-tx for drop/type-change; 'english' pin; scope-parity search with rank+id ordering; search_tsv strip; blocklist growth; UAC-2.1 arc incl. add-field-reindex clause; the trgm-toggle debt landing.

### v2p6 webhooks — reviewed in full

- **F-v2p6-1 (MEDIUM)** — Delivery claim semantics are underspecified: `ClaimDueDeliveries … FOR UPDATE SKIP LOCKED` only holds locks inside a transaction, but delivery then does seconds of HTTP before `MarkDelivered/MarkRetry` — the plan neither holds the tx (bad: HTTP inside tx) nor marks rows in-flight. Single-worker reality: rows aren't actually claimed; a crash between deliver and mark re-sends (at-least-once) without attempt increment. Fix: drop the misleading FOR UPDATE SKIP LOCKED (plain SELECT), state **at-least-once delivery** explicitly (consumers dedupe on `X-CMS-Delivery`), and document the crash-window re-send.
- **F-v2p6-2 (MEDIUM, doc decision)** — **Unpublish emits no event** (F-22's set is create/update/publish/trash), yet the 04 amendment invites consumers to lengthen cache TTLs on the publish purge signal — an unpublished record then stays in consumer caches indefinitely. Triage: either amend F-22/spec to add `record.unpublish` (requirements touch) or the 04 amendment must scope TTL-lengthening advice to publish/trash-driven invalidation and name the unpublish hazard.
- **F-v2p6-3 (LOW)** — `netip` classification misses shared address space `100.64.0.0/10` (CGNAT); add it (and consider 192.0.0.0/24) to the explicit deny list in `safeTransport`.
- **F-v2p6-4 (LOW)** — Add one consumer-guidance sentence to the 04/05 amendment: verify HMAC over `timestamp.body` AND reject stale `X-CMS-Timestamp` (replay window).

Verified-good (v2p6): outbox-in-tx atomicity with rollback test; nudge+ticker meeting the 30s bound; seal refactor with V1-suite-stays-green gate; SSRF matrix incl. DNS-pinning and redirect refusal; retry ladder + terminal failed; secrets regex growth; TEST-NET e2e trick; BR-HOOK-1/2 texts with enforcement points; retention duty 9.

### v2p7 m2m-relations — reviewed in full

- **F-v2p7-1 (MEDIUM)** — Publish/restore reconciliation hard-fails when a snapshot's membership target was **purged** since the draft (FK 23503 on the reconcile INSERT → the whole publish/restore 422s). EC-5's drift philosophy says restore degrades gracefully. Fix: reconcile filters the array to still-existing targets and reports dropped ids in the audit detail (mirror `skipped` from drift), instead of failing the operation.
- **F-v2p7-2 (LOW)** — Add a self-referencing M2M test case (`posts.related → posts`): CASCADE-own-memberships + RESTRICT-inbound on one table is subtle enough to pin.

Verified-good (v2p7): freeze-contract arc test; join anatomy/naming/truncation + collision check; rename cascade incl. target-rename-touches-nothing; omit-not-null with the leak rationale; page-wide query batching; 500-cap; cardinality conversion rejection; TxHook composition; purge mappings.

### v2p8 import-export-gdpr — reviewed in full

- **F-v2p8-1 (HIGH, carry-in confirmed)** — Task 4's erasure transaction deletes `cms_end_users` FIRST, then `cms_refresh_tokens` — but `cms_refresh_tokens.end_user_id` carries an FK (NO ACTION, v1p1 DDL): the parent delete violates it immediately, so **erasure always fails for any user who ever had a refresh token**. Fix: children first (refresh, reset), user row last (existence-check up front for ErrNotFound); V3's cart/entitlement CASCADEs stay correct either way. Self-detecting via the BR-AUTH-15 test, but the plan prescribes the broken order.
- **F-v2p8-2 (MEDIUM)** — `RedactRevisions` validates field names against the **current** snapshot, so PII in revisions under renamed/dropped field slugs can never be redacted (unknown → 422) — the exact scenario GDPR redaction exists for. Fix: validate shape only (legal slug grammar) or current-∪-historical keys; keep the audit detail.
- **F-v2p8-3 (dependent on F-v1p3-4 below)** — Schema-import pass 2 re-creates fields via `AddField`, which under the V1 whitelist provisions NO indexes: imported `unique`/`indexed`/`trgmIndexed` fields lose their indexes silently, breaking the UAC-2.2 "identical definitions" claim at the physical level (the export diff compares configs, masking it). Resolved by the F-v1p3-4 fix.

Verified-good (v2p8): slug-keyed export with no UUIDs; two-pass cycle dissolution; strict-validate/per-op-apply with honest non-transactionality; ForceID additive flag; forward-reference drop-and-report stance; NDJSON streaming (needs F-v1p1-2's Flusher fix); tombstone literal; redaction never touches cms_audit_log; V2 gate sweep incl. doc future-tense scan.

### V1 ADDENDUM (found during V2 review)

- **F-v1p3-4 (HIGH, doc + plan)** — The DDL whitelist cannot provision declared indexes for fields added AFTER collection creation: 03's `AddField` template is bare `ADD COLUMN`; `AddIndex` creates only the plain composite `(field, id)` form; **no whitelisted op creates the partial-unique index** 07 contracts for `unique` fields (and V2-P4's trgm toggle is separate). A field added later with `unique: true` stores the flag but never gets enforcement — silent integrity gap; same for `indexed` unless the UI orchestrates AddIndex, and import (F-v2p8-3) hits it wholesale. Fix: `AddField` emits the column PLUS its declared index DDL (composite/partial-unique/trgm) in the same tx, 03 amendment updates the template row; `AddIndex/DropIndex` remain the standalone toggles.

### V2 cross-cutting pass — complete

Migrations 0003–0008 consistent; waiver seed/empty choreography correct modulo F-v2p1-1; vocabulary growth enforced by v2p1's parser-equality test in every phase; UAC-2.1→p5, 2.2→p6+p8, 2.3→p2, 2.4→p1+p3 all anchored; interface hand-offs (Actions(), TxHook→p7, JunctionArrays→p8, trgm-UI debt p4→p5) verified; V2's "V1 gates stay green" wording inherits F-v1x-1's 1.6/1.7 enumeration fix.

**Status: V2 review COMPLETE (8/8 + cross-pass). V2 findings: 1 HIGH carried+confirmed (F-v2p8-1) + F-v2p1-1 HIGH (self-detecting) + 1 new V1 HIGH (F-v1p3-4); 8 MEDIUM; 8 LOW. Next: V3 plans (5).**

### v3p1 commerce-foundation — reviewed in full

- **F-v3p1-1 (MEDIUM)** — Price-field protection covers drop/type-change but not **rename**: `commerce_config.priceField` stores a slug, so renaming the price field strands the config (cart hydration/checkout then error at runtime). Fix jointly with F-v2p5-1: one rename-propagation mechanism updating `search_config` AND `commerce_config` slugs in the rename tx.
- **F-v3p1-2 (LOW)** — Task 4's `grep -c '(V3)' docs/BUSINESS_RULES.md` false-positives on the preamble sentence ("Rules tagged (V2) or (V3)…"); grep the rule-tag pattern instead.

### v3p2 carts — reviewed in full

- **F-v3p2-1 (MEDIUM)** — Cart item ops are read-modify-write over the `items` JSONB with no concurrency control: two concurrent AddItems lose one (last-write-wins upsert). Fix: wrap ops in a tx with `SELECT … FOR UPDATE` on the cart row (`GetCartByUserForUpdate`).
- **F-v3p2-2 (LOW)** — The `display` value (first `text` field) ignores audience field rules — a field `hideFrom: ["end_user"]` still leaks via cart hydration; pick the first *visible* text field.

### v3p3 stripe-orders — reviewed in full

- **F-v3p3-1 (HIGH, money correctness)** — `checkout.session.completed → paid` is unconditional, but for **async payment methods** Stripe fires `completed` with `payment_status: "unpaid"`, then `async_payment_succeeded/failed` later. As planned: an async order is marked `paid` at completed, and the later `async_payment_failed` no-ops (`pending→failed` matches 0 rows) — **order paid, money never received**. Fix: read `payment_status` from the completed event (`paid` → transition; `unpaid` → stay pending), handle `checkout.session.async_payment_succeeded → paid`; optionally pin `payment_method_types=card` at session create as belt-and-suspenders.
- **F-v3p3-2 (LOW)** — "clear the user's cart" inside the paid tx must go through a tx-scoped store call (`DeleteCartByUser` via `q.WithTx`), not `CartService.Clear` — name it.

Verified-good (v3p3): BR-COM-1 type-shape enforcement (no amount field exists); signature scheme with rotation + tolerance + constant-time; zero-rows idempotency incl. replay/hook-not-rerun tests; pending-only cleanup vs webhook race; amount-mismatch warn line; secrets regexes; manual gate step verbatim.

### v3p4 video-paywalls — reviewed in full

- **F-v3p4-1 (MEDIUM)** — The paywall is only real if the CF Stream video has `requireSignedURLs` enabled — otherwise the public unsigned URL plays regardless of our tokens. Add the operator prerequisite to the 09/05 amendments and to the manual gate step ("confirm requireSignedURLs on the test video").
- **F-v3p4-2 (MEDIUM, product decision)** — Unpublishing (or purging) a purchased record locks out entitled buyers: the token endpoint's step-2 visibility gate fires before the entitlement check. Triage: accept + document ("unpublish suspends buyer playback"), or let a valid entitlement bypass the publish floor at step 2 for the token route only.

Verified-good (v3p4): field type end-to-end with no-URL-anywhere regex test; CF JWT format (kid/sub/exp RS256) matches CF Stream's scheme; TTL cap at the adapter; idempotent hook+backfill; gate-order normative list; erasure CASCADE assertion; PEM secrets regex.

### v3p5 tiptap-blocks — reviewed in full

- **F-v3p5-1 (LOW)** — Gate task arithmetic typo: "seventeen + thirteen = twenty-two plans" (it's 9+8+5=22; 17+13=30).
- **F-v3p5-2 (LOW)** — Add a test case: a block inside an *expanded relation target's* richText also resolves (the shared-walk assumption, currently untested).

Verified-good (v3p5): shape-only validation with survival semantics; batched anonymous-strict resolution with deep-equal null-survival test; pure-renderer figure placeholders; missing-reference editor state; shared-allowlist hardening note; gate checklist covers §C incl. both manual steps + mode matrix.

### V3 cross-cutting pass — complete

Migrations 0009–0012 consistent; waiver stays empty (all new BRs land with tests); UAC-3.1→p3, 3.2→p4, 3.3→p5 anchored; erasure accrual (carts CASCADE, orders tombstone, entitlements CASCADE) coherent given F-v2p8-1's reorder; OrderPaidHook/CommerceConfig/RequireCommerce hand-offs verified; mode matrix in the gate.

**Status: REVIEW COMPLETE — all 22 plans + 3 cross-cutting passes. Totals: 10 HIGH, 20 MEDIUM, ~25 LOW. Consolidated triage report: `docs/reviews/2026-07-14-plan-review-v1-v3.md`.**

## Cross-version ledger

### V1 → V2/V3 (what later plans consume; verified as-produced unless flagged)

- `audit.Sink/Recorder/Emit`, `Event.Detail["request_id"]` promotion — v2p1 fanout slots in cleanly.
- `query.EncodeCursor` is UUID-typed — v2p1 correctly plans a local bigint codec.
- `trace.sh`: structural + `\(V[23]\)` exemptions, waiver-match regex `^${id}[[:space:]]` (**F-v1p1-4**: bare `BR-LIFE-9` reseed in v2p1 won't match — fix regex or annotate the entry).
- `statusWriter` lacks Flush/Unwrap (**F-v1p1-2**) — v2p8 NDJSON streaming breaks without the fix.
- Seven → eight → nine system columns (v2p2 `publish_at`, v2p3 `seo`); v2p2's DO-block backfill index `ix_c_<slug>_publish_at` can exceed 63 bytes at V1's 55-char slug cap (**file as F-v2p2-1**: apply the truncation helper's rule inside the migration or cap differently).
- `cms_refresh_tokens.end_user_id` has an FK (NO ACTION) — **v2p8's erasure transaction must delete child rows BEFORE `cms_end_users`** (its Task 4 lists the user-row delete first — file as F-v2p8-1 HIGH: reorder or the tx always fails).
- `content.Set(ctx, snap, col, rules, input, mv)` final signature (F-v1p4-3 cleans the stated duplicate); merge-base fix (F-v1p4-1) affects v2p3's `$seo` snapshot plumbing the same way — verify when fixing.
- `Save(..., initialStatus)` (P6 additive) + `SaveOpts{ForceID}` (v2p8 additive) — two variadic/positional growths on the same method; reconcile in one shape at fix time.
- Relation FKs are optional (03: `AddForeignKey` is explicit) — v2p7/v3p2 wording "FK enforces existence" needs softening.
- V1 e2e = `make e2e` (P9 Task 10); V2/V3 sweeps reference it correctly, but say "UAC-1.1…1.6" (F-v1x-1).

**Status: V1 review COMPLETE (9/9 plans + cross-pass). Finding count: 7 HIGH (F-v1p1-1, F-v1p1-3, F-v1p1-4, F-v1p3-1, F-v1p4-1, F-v1p5-1, F-v1p8-1), 9 MEDIUM (F-v1p2-1, F-v1p4-2, F-v1p4-3, F-v1p5-2, F-v1p5-3+F-v1p9-2, F-v1p7-1, F-v1p8-2, F-v1p9-1, F-v1x-1), 12 LOW. Next: V2 plans (8) — carry-in findings F-v2p1-1, F-v2p2-1, F-v2p8-1 already identified from V1 interactions.**
