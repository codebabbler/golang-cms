# P6 Public API Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The public content surface: `access.Evaluator` over the 12-access-rules grant matrix, `/api/v1/collections/{slug}/records` CRUD under Decisions, the public filter grammar, ETag/304 caching, CORS, anonymous rate limits, and idempotent creates — so UAC-1.2 passes: publish → unauthenticated fetch returns the record.

**Architecture:** Implements `12-access-rules.md` in full (schema §1, algorithm §3, validation §4, audiences §5, key scopes §6), 04's caching/CORS/idempotency/expansion, BR-API-2/4/5/6/7 and BR-RBAC-2/3/4/6/7 + BR-LIFE-6. Every handler: `Decide` → `query.Builder.WithDecision` → `Document.Set` for writes — the P4 seams make bypass unrepresentable.

**Tech Stack:** No new dependencies. sqlc over `cms_idempotency_keys`.

> **Authored before execution (D-V1-3 amendment):** re-validate at execution start; docs win. Invoke `query-builder-invariants`, `api-conventions`, and `sqlc-workflow` before their tasks.

## Global Constraints

- Grant-matrix schema verbatim (12 §1): actions `read create update delete publish`; grant keys `minRole minRoleOwn endUsers anonymous createStatus` only; lattice `viewer < contributor < editor < admin < super_admin`; omitted action ⇒ deny for governed classes; `super_admin`/`admin` implicit full grant on content actions; publish floors at `editor` regardless of rules (BR-LIFE-3).
- The three worked examples of 12 §2 are normative — the evaluator test suite encodes all three.
- Fail-closed (12 §4, N-11): unparseable `access_rules` ⇒ `Decision{Allowed:false}` for every request until corrected.
- `createStatus` (12 §1/§3): applies to `end_user`/`api_key` creates only; default `draft`; `published` create writes live row published + first revision `published=true` in one tx (BR-LIFE-2) and audits `createStatus` in detail. Admin-kind creates always start `draft`.
- Owner-draft visibility (BR-API-2): ScopePublic includes the authenticated public principal's own records regardless of status (P4 builder already compiles it); expansion targets are strictly published-only even for owners.
- Public filter grammar (BR-API-4): indexed/unique fields only, ops `eq neq lt lte gt gte in` (no `contains`), `in` comma-separated; violations → `422` naming the offender.
- Caching (BR-API-5): anonymous ScopePublic GETs → `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` + strong `ETag` + `If-None-Match` → `304`; ANY credentialed request → `no-store`, no exception; both → `Vary: Authorization, Cookie`.
- CORS (BR-API-6): `/api/v1` only; `Access-Control-Allow-Origin: *` on every response; preflight `OPTIONS` handled **before** auth and rate limiting, answered with `Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key`, `Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS`, `Access-Control-Max-Age: 86400`; non-preflight responses add `Access-Control-Expose-Headers: X-Request-ID, ETag`. `/api/admin/*` and the SPA emit no CORS headers.
- Rate limits (BR-API-7): anonymous public reads 300 req/min per client IP → `429`; `?count=exact` requires an authenticated principal — anonymous → `422 validation_failed` naming the parameter.
- Idempotency (04 §Idempotent Creates): optional `Idempotency-Key` ≤ 128 bytes on public/API-key creates; row inserts in the record's transaction; unique `(key_hash, principal_id)`; concurrent duplicate waits up to **5 s** — committed → replay original, aborted → fresh create, still in flight → `409` retry; same key + different `request_hash` (sha256 of raw body) → `422`; first → `201`, replay → `200` current representation, purged → `404`.
- Expansion (04, EC-7): `?expand=<relationField>` one level; one batched `IN` query per field; trashed/draft/hidden targets → `null` (BR-LIFE-6); unexpanded relations serialize as bare UUID.
- Body caps: record create/update 5 MiB; everything else 64 KiB.
- Waiver shrink: `BR-API-2 4 5 6 7`, `BR-RBAC-2 3 4 6 7`, `BR-LIFE-6`.
- `WriteError` only; commits plain; branch `main`.

## File Structure

```
internal/access/rules.go             grant parsing + ValidateAccessRules (12 §1/§4)
internal/access/rules_test.go
internal/access/evaluator.go         Decide (12 §3) + field-rule audiences (§5)
internal/access/evaluator_test.go
internal/httpapi/cors.go             CORS middleware (BR-API-6)
internal/httpapi/cors_test.go
internal/httpapi/etag.go             strong ETag + 304 + BR-API-5 header logic
internal/httpapi/etag_test.go
internal/httpapi/public_records.go   /api/v1/collections/{slug}/records handlers
internal/httpapi/expand.go           relation expansion (EC-7)
internal/httpapi/idempotency.go      key handling around create
internal/httpapi/public_records_integration_test.go
internal/store/queries/idempotency.sql
internal/httpapi/admin_collections.go (modify) rule write-validation (BR-RBAC-7)
internal/httpapi/ratelimit.go        (modify) anonymous 300/min bucket
internal/app/app.go                  (modify) wire evaluator
docs/trace-waivers.txt               (modify) shrink
```

---

### Task 1: Grant parsing and the Evaluator

**Files:**
- Create: `internal/access/rules.go`, `internal/access/evaluator.go`
- Test: `internal/access/rules_test.go`, `internal/access/evaluator_test.go`

**Interfaces:**
- Produces:
  - `access.Grant{MinRole, MinRoleOwn *Role; EndUsers string; Anonymous bool; CreateStatus string}`; `access.Rules map[string]Grant` (action-keyed); `access.ParseRules(raw json.RawMessage) (Rules, error)` — strict decode, unknown keys error (fail-closed feeds off this).
  - `access.Action` — `ActionRead ActionCreate ActionUpdate ActionDelete ActionPublish`.
  - `access.RecordMeta{CreatedBy uuid.UUID; Status string}`.
  - `access.NewEvaluator() *Evaluator`; `(*Evaluator).Decide(p Principal, col *schema.Collection, action Action, record *RecordMeta) Decision` — the 12 §3 order:
    1. admin-kind with role ≥ admin → `Decision{Allowed: true, Principal: p}` (+field rules — hidden for non-super_admin audiences per §5).
    2. admin-kind below admin: `role ≥ minRole` → unrestricted; else `role ≥ minRoleOwn` → `PredicateOwnerOnly`; else deny.
    3. end_user: per `endUsers` (`own` ⇒ owner predicate; reads ride ScopePublic's owner-draft form from P4).
    4. anonymous: `read` + `anonymous:true` only.
    5. api_key: scopes contain `(col.ID, action)`; reads published-only unless `DraftAccess` — expressed as `Decision.DraftAccess bool` the handler maps to scope choice; revoked keys arrived as anonymous already (P5).
    6. field rules per §5 → `FieldRules`.
    - `ParseRules` failure ⇒ `Decision{Allowed:false}` for every principal/action on that collection (12 §4).
    - Publish: after step 2, deny unless the resolved role ≥ editor (floor, BR-LIFE-3); `end_user`/`anonymous`/`api_key` never get publish. Unpublish evaluates as publish.
  - `(*Evaluator).CreateStatus(col *schema.Collection, p Principal) string` — `end_user`/`api_key` → grant's `createStatus` (default `"draft"`); admin kinds → `"draft"` always.
  - `access.ResolveAudience(p Principal) string` — the §5 audience name (`super_admin…viewer`, `end_user`, `anonymous`, `api_key`).

- [ ] **Step 1: failing tests** — the three 12 §2 worked examples end-to-end as `TestBR_RBAC_2_WorkedExamples` (blog: anonymous read allowed, contributor create denied; comments: end_user create allowed with `createStatus=published`, contributor update ownerOnly, editor update unrestricted; ingestion: anonymous read denied, admin implicit create); `TestBR_RBAC_3_DefaultDenyAndImplicitGrant` (omitted action denies editor; admin passes); `TestBR_LIFE_3_PublishFloor` (minRole contributor on publish still denies contributor; end_user never); `TestBR_RBAC_7_FailClosedOnMalformedRules` (junk JSONB → every Decide false); `TestBR_RBAC_4_AudienceFieldRules` (hideFrom `["anonymous"]` hides for anonymous, visible for editor; readOnlyFor rides into FieldRules); `TestCreateStatusResolution` (api_key honors matrix, admin always draft — 12 §6).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: access evaluator — grant matrix, default deny, publish floor (BR-RBAC-2/3, BR-LIFE-3)"`

---

### Task 2: Rule write-validation in the admin API

**Files:**
- Modify: `internal/access/rules.go` (ValidateAccessRules), `internal/httpapi/admin_collections.go` (call it on create + rule update; add `PUT /api/admin/collections/{slug}/access-rules`)
- Test: extend `rules_test.go` + `admin_collections_integration_test.go`

**Interfaces:**
- Produces: `access.ValidateAccessRules(raw json.RawMessage) []RuleError` where `RuleError{Path, Message string}` — the eight 12 §4 rejections verbatim: unknown action key; unknown grant key; unknown role name; `minRoleOwn > minRole`; `anonymous` outside `read`; `createStatus` outside `create`; non-enum `createStatus`; non-boolean `anonymous` / non-enum `endUsers`. Inert-but-legal shapes (publish grants below the floor) validate clean (12 §4 last line).
- Handler: `422 validation_failed` with `details:[{path, message}]`.

- [ ] **Step 1: failing tests** — one case per rejection path (`TestBR_RBAC_7_WriteValidationRejects…` umbrella, eight subtests) + inert-publish-grant accepted; HTTP-level: `PUT .../access-rules` with `{"read":{"minRole":"emperor"}}` → 422 naming `read.minRole`.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: closed grant-matrix write validation (BR-RBAC-7)"`

---

### Task 3: CORS, anonymous rate limit, count=exact gate

**Files:**
- Create: `internal/httpapi/cors.go`
- Modify: `internal/httpapi/ratelimit.go`, `internal/httpapi/router.go`
- Test: `internal/httpapi/cors_test.go`, extend `ratelimit_test.go`

**Interfaces:**
- Produces:
  - `httpapi.CORS(next http.Handler) http.Handler` — mounted on the `/api/v1` subtree ONLY, **outside** RateLimit and Auth so preflights bypass both (BR-API-6): `OPTIONS` + `Access-Control-Request-Method` present → `204` with the three preflight headers + `Access-Control-Max-Age: 86400`, short-circuit; otherwise set `Access-Control-Allow-Origin: *` + `Access-Control-Expose-Headers: X-Request-ID, ETag` and pass through.
  - RateLimit gains the anonymous-read bucket: on `/api/v1` GETs with an anonymous principal → `ip-read:<addr>` at **300/min** (BR-API-7). (Principal must be resolved first — so this check lives AFTER Auth in the chain as a distinct `PublicReadLimit` middleware; the pre-auth RateLimit slot keeps the login/register buckets. 04's order `RateLimit → Auth` is preserved for the credential buckets; the anonymous bucket is principal-dependent by definition — flagged as the documented interpretation.)
  - `ParsePagination` gains the gate: `count=exact` + anonymous → `422` naming `count`.

- [ ] **Step 1: failing tests** — preflight OPTIONS: 204, all three headers + max-age, and it passes even when the IP is rate-limit-saturated (exemption proof); every /api/v1 response carries ACAO:*; expose-headers present on GET; `/api/admin/*` response carries NO CORS headers; 301st anonymous read in a minute → 429 (`TestBR_API_7_AnonymousReadLimit`); authenticated reads bypass that bucket; anonymous `?count=exact` → 422 naming the parameter.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: wildcard CORS with exempt preflights; anonymous read limits (BR-API-6/7)"`

---

### Task 4: Public reads — list/get, filter grammar, ETag, cache headers

**Files:**
- Create: `internal/httpapi/public_records.go`, `internal/httpapi/etag.go`
- Modify: `internal/httpapi/router.go`
- Test: `internal/httpapi/etag_test.go`, `internal/httpapi/public_records_integration_test.go` (read half)

**Interfaces:**
- Produces:
  - `GET /api/v1/collections/{slug}/records` — Decide(read) → deny → `404 not_found` for anonymous (existence-hiding) / `403` for authenticated principals; builder `ScopePublic` + Decision; filters parsed by P4's `ParseFilters` restricted to the public grammar (indexed/unique + no contains — the builder enforces; handler maps builder errors to `422`); pagination public = capped offset only (`cursor` → `422` in V1 public — F-27 is V2); `count=exact` per Task 3 gate; serialization strips `FieldRules.Hidden` fields and system column `deleted_at`.
  - `GET /api/v1/collections/{slug}/records/{id}` — same scope; 404 when invisible.
  - `httpapi.WriteCached(w, r, p access.Principal, body []byte)` — anonymous: `Cache-Control: public, s-maxage=60, stale-while-revalidate=60`, strong `ETag: "<sha256-16hex>"` over the exact body, `If-None-Match` match → `304` (headers only); credentialed: `Cache-Control: no-store`; both: `Vary: Authorization, Cookie` (BR-API-5).
- [ ] **Step 1: failing tests** — unit (etag): anonymous GET carries s-maxage+ETag+Vary; matching If-None-Match → 304 empty body; JWT-bearing request → no-store (never s-maxage); integration: **UAC-1.2** — publish via admin, fetch unauthenticated → 200 with the record (`TestUAC_1_2_PublishedRecordPubliclyReadable` with `t.Run("BR-API-2 draft invisible anonymously")` before publish, owner-draft: creator end_user sees own draft, stranger end_user does not); filter on unindexed field → 422; `contains` publicly → 422 (`TestBR_API_4_PublicGrammar`); offset 10_001 → 422.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: public reads with ETag/304 and the BR-API-5 cache contract (UAC-1.2)"`

---

### Task 5: Relation expansion

**Files:**
- Create: `internal/httpapi/expand.go`
- Test: extend `public_records_integration_test.go`

**Interfaces:**
- Produces: `httpapi.ExpandRelations(ctx, snap, col, records []map[string]any, fields []string, deps ExpandDeps) error` — for each named relation field: collect UUIDs → ONE builder query per field (`ScopePublic`, **anonymous-strict form** — owners never see their own drafts through expansion, 04) with `Where(id, OpIn, ids)` → replace values: found+published → embedded object (hidden fields stripped per the TARGET collection's field rules for this audience); missing/trashed/draft → `null` (BR-LIFE-6, EC-7). Non-relation `expand` names → `422`. `ExpandDeps` carries the pool/queries handle the builder path needs.
- [ ] **Step 1: failing tests** — `TestBR_LIFE_6_EC7_ExpansionNullsInvisibleTargets`: A relates to B(published) and C(draft) and D(trashed) → expand yields object, null, null; owner requesting own list still gets null for own draft target (owner-draft never crosses expansion); query-count assertion: 30 records expand with exactly one extra query (pgx tracer counting).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: one-level relation expansion, invisible targets null (BR-LIFE-6, EC-7)"`

---

### Task 6: Public writes — createStatus, idempotency, update/delete

**Files:**
- Create: `internal/httpapi/idempotency.go`, `internal/store/queries/idempotency.sql`
- Modify: `internal/httpapi/public_records.go`, `internal/lifecycle/service.go` (Save gains `initialStatus string` — `createStatus` support, BR-LIFE-2 one-tx published create)
- Test: extend `public_records_integration_test.go`

**Interfaces:**
- sqlc: `InsertIdempotencyKey` (in-tx), `GetIdempotencyKey(key_hash, principal_id)`.
- Produces:
  - `POST /api/v1/collections/{slug}/records` — Decide(create) → `Document.Set` (field rules for the audience) → status = `Evaluator.CreateStatus(col, p)` → `lifecycle.Save(..., initialStatus)`; a `published` create writes live row + first revision published in ONE tx (12 §3, BR-LIFE-2); audit `content.record.create` with `createStatus` detail. Anonymous creates: only if... 12 §1 gives `create` no `anonymous` key → anonymous create always denied (`404`). 5 MiB body cap on this route class.
  - Idempotency wrapper: header absent → plain create; present (≤128 bytes else `422`) → `key_hash = sha256(key)`, `request_hash = sha256(raw body)`; try `GetIdempotencyKey`: found + same request_hash → load record → `200` current representation (`404` if purged); found + different hash → `422`; not found → create with `InsertIdempotencyKey` inside the same tx; unique-violation on insert → someone racing: poll `GetIdempotencyKey` up to **5 s** (50 ms interval — the 09 "Idempotency duplicate-wait bound") → resolved → replay; still absent (first tx still open) → `409 conflict` "request in progress — retry".
  - `PUT .../{id}` — Decide(update) (+predicate row-gates via builder-scoped fetch first); version-checked like P4. `DELETE .../{id}` — Decide(delete) → `lifecycle.Trash` (soft — BR-LIFE-4).
- [ ] **Step 1: failing tests** — createStatus arc (`TestBR_LIFE_2_CreateStatusPublishedIsOneTransaction`: comments-style rules; end_user create → immediately anonymous-readable; revision 1 has published=true); idempotency: same key+body twice → one record, second response 200; different body → 422; concurrent same-key (two goroutines) → exactly one record, loser either replays or 409s (`TestIdempotentCreates`); ownerOnly update: end_user edits own → 200, other's → 404/403 per predicate; api_key with `draftAccess:true` lists drafts, without → published-only (12 §6).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: public writes — createStatus creates and idempotency keys (BR-LIFE-2, 04 §Idempotent Creates)"`

---

### Task 7: Wiring, replacement of P4 interim floors, smoke, waiver shrink

**Files:**
- Modify: `internal/app/app.go` (wire Evaluator into RouterConfig), `internal/httpapi/admin_records.go` (replace the P4 `// P6:` interim `RequireRole` floors with real `Decide` calls for sub-admin roles), `docs/trace-waivers.txt` (remove BR-API-2,4,5,6,7 + BR-RBAC-2,3,4,6,7 + BR-LIFE-6)
- Test: extend `internal/app/smoke_integration_test.go`; run the 09 cache-contract smoke-check runbook shape as a test.

- [ ] **Steps:** grep `// P6:` and replace every interim floor (admin records handlers now Decide per action — contributors get ownerOnly where the matrix says so); smoke leg = full UAC-1.2 arc via curl-shaped client (setup → login → collection → record → publish → anonymous fetch 200 → If-None-Match 304 → JWT fetch no-store); `make test && make trace && make lint` green (waived −11); commit — `git commit -m "feat: P6 smoke green — public API complete (UAC-1.2)"`.

## Plan Self-Review Notes

- The anonymous 300/min bucket runs post-Auth (principal-dependent) while credential buckets stay pre-Auth — this reading of 04's `RateLimit → Auth` order is flagged for execution review; BR-API-7's semantics are unambiguous even if the slot placement has two readings.
- `lifecycle.Save` gaining `initialStatus` is an additive signature change to a P4-internal service (not a 02 contract) — the P4 plan's `Save` already threads status internally; execution reconciles.
- Anonymous deny renders `404` (existence-hiding) vs authenticated deny `403` — 04/12 don't pin this split; flagged as a decision, consistent with the threat model's enumeration posture.
- Media-field delivery-URL resolution at read time is P7's; P6 serializes media values as bare UUIDs (no media rows exist yet).
