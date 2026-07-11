# Round-2 Architecture Review Remediation — Design

**Date:** 2026-07-11 · **Status:** Approved (owner, 2026-07-11) · **Source review:** `docs/reviews/architecture-review-round2-2026-07-11.md` (AR2-1…AR2-27)

This is a **documentation-only** remediation pass (implementation has not started). It resolves all 27 round-2 findings across `docs/BUSINESS_RULES.md`, `docs/REQUIREMENTS.md`, `docs/architecture/*`, `docs/api/openapi.yaml`, and appends a Resolution Status table to the round-2 review. Where this spec quotes rule text, that text is normative and must land verbatim (allowing only surrounding-sentence stitching).

## Decision Log (all options accepted as recommended, 2026-07-11)

| ID | Decision | Resolves |
|---|---|---|
| D2-1 | `createStatus` grant key + owner-draft visibility | AR2-1 |
| D2-2 | Wildcard CORS on `/api/v1`; no CORS on `/api/admin` | AR2-2 |
| D2-3 | Instance-lock watchdog + bounded startup retry | AR2-3 |
| D2-4 | Media deletion outbox (`cms_media_deletions`) | AR2-4 |
| D2-5 | Argon2id admission semaphore | AR2-5 |
| D2-6 | Public `contains` removed in V1; pg_trgm path in V2 | AR2-6 |
| D2-7 | Medium sweep as specified in §7 (incl. registration gate default **disabled**, env table 16→17) | AR2-7…AR2-14 |
| D2-8 | Low sweep as specified in §8 | AR2-15…AR2-27 |

---

## 1. Public-Create Lifecycle (D2-1, AR2-1)

### 1.1 `createStatus` grant key

`12-access-rules.md` §1 grant-object table gains one row:

| Key | Type | Meaning |
|---|---|---|
| `createStatus` | `"draft"` \| `"published"` | Valid on `create` only. The initial `status` of records created by `end_user` or `api_key` principals; default `draft`. Never applies to admin-kind principals — their creates always start as `draft` and go live only through the `publish` action. |

Normative semantics (12 §3, new paragraph after the publish-floor paragraph):

> For `action = create` by an `end_user` or `api_key` principal, the created record's initial `status` is the grant's `createStatus` (default `draft`). `createStatus: "published"` is initial state, not a `publish` action: BR-LIFE-3's editor floor governs the `publish` action only, and because `createStatus` never applies to admin-kind principals, no staff role can use it to bypass that floor. A `createStatus: "published"` create writes the live row as published and its first revision with `published = true` in the same transaction (BR-LIFE-2), and emits `content.record.create` with `createStatus` in the audit detail.

12 §4 validation gains two bullets:

- `createStatus` present on any action other than `create`, or
- a non-enum `createStatus` value (outside `draft`, `published`).

12 §2 comments example: the `create` grant becomes `{ "endUsers": "all", "minRole": "contributor", "createStatus": "published" }`, with a one-line comment that comments go live on creation while the other examples retain the moderation-queue default.

### 1.2 Owner-draft visibility

**BR-API-2 (amended, verbatim):**

> **BR-API-2.** Public and API-key reads return only published, non-trashed records unless the key scope explicitly grants draft access or the record was created by the requesting principal: authenticated public principals (end users and API keys) always see records they created, including drafts (owner-draft visibility). Anonymous reads remain published-only without exception.
> *Enforcement:* `query.Builder` public-scope base predicate.

`02-core-interfaces.md` builder invariant 3 becomes:

> 3. `ScopePublic` appends `status = 'published'` (BR-API-2), relaxed to `(status = 'published' OR created_by = <principal>)` when the Decision carries an authenticated public principal (owner-draft visibility); anonymous requests get the strict form.

Caching note (04 §Caching): owner-draft-widened responses are always credentialed and therefore already `no-store` under BR-API-5; the anonymous cacheable class is unaffected.

Relation expansion (04 §Relation Expansion) adds one sentence:

> Expansion targets resolve strictly published-only even for their owners — owner-draft visibility applies to the requested records, never to expanded targets.

### 1.3 Ripples

- `07-data-model.md` Live-Table/Revisions Contract adds: "Records created through the public API take the collection's `createStatus` (`12-access-rules.md` §1): a `published` create writes the live row as published and the first revision with `published = true` in one transaction."
- **BR-LIFE-2 (amended)** appends: "Records created with `createStatus: "published"` (`12-access-rules.md`) start published: the create transaction writes the live row as published and marks the first revision published."
- REQUIREMENTS F-12 appends: "Collections may set `createStatus` to publish public-API creates immediately (12-access-rules.md §1)."

## 2. CORS (D2-2, AR2-2)

**BR-API-6 (new, verbatim):**

> **BR-API-6.** Every `/api/v1` response carries `Access-Control-Allow-Origin: *`. Preflight `OPTIONS` requests are handled before authentication and rate limiting and answered with `Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key`, `Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS`, and `Access-Control-Max-Age: 86400`; non-preflight responses expose `X-Request-ID` and `ETag` via `Access-Control-Expose-Headers`. `/api/admin/*` and the SPA emit no CORS headers — the admin surface is same-origin only (cookie + CSRF). The wildcard is safe because bearer tokens are not ambient credentials: CORS never causes a browser to attach a victim's JWT, and cookies are never accepted on `/api/v1`.
> *Enforcement:* `middleware.CORS` mounted on the `/api/v1` subtree only.

`04-api-layer.md` gains a "## CORS (BR-API-6)" section restating this with one addition: because `Access-Control-Allow-Origin` is the constant `*`, cached anonymous responses under BR-API-5 need no `Vary: Origin`. `05-auth-security.md` §6 threat model gains: "Cross-origin abuse of the admin surface is prevented by the absence of CORS headers on `/api/admin/*` plus cookie `SameSite=Lax` and CSRF (BR-API-6)."

## 3. Instance-Lock Hardening (D2-3, AR2-3)

**BR-RUNTIME-8 (amended, verbatim):**

> **BR-RUNTIME-8.** Exactly one process serves at a time: before opening the listener, the binary acquires a process-lifetime advisory lock (session-scoped, held on a dedicated connection with TCP keepalives, distinct from the migration and schema keys). Startup retries acquisition with backoff for up to 120 seconds — riding out a crashed predecessor's lingering session — then fails with a clear log line. If the lock connection drops at any point after acquisition, the process exits non-zero: serving without the lock is never permitted (fail closed, N-11). *(Resolves EC-16.)*
> *Enforcement:* `app.Run` — `pg_advisory_lock` on the instance key, dedicated connection, watchdog on connection loss.

`09-deployment.md` startup step 3 updates to match (bounded retry replaces "fails startup immediately"; the "stop-then-start remains the only supported upgrade path" sentence stays). 09 §Runbooks gains Postgres keepalive guidance: set `tcp_keepalives_idle=60`, `tcp_keepalives_interval=10`, `tcp_keepalives_count=3` (or `idle_session_timeout` on managed Postgres) so a dead process's session releases the instance lock within ~2 minutes, inside the startup retry window. `01-system-overview.md` lifecycle summary notes the watchdog in one clause.

## 4. Media Deletion Outbox (D2-4, AR2-4)

**BR-MEDIA-5 (new, verbatim):**

> **BR-MEDIA-5.** Media deletion runs one transaction that deletes the `cms_media` row — FK RESTRICT produces the `409` while any record references it — and inserts the `object_key` into `cms_media_deletions`; the storage object is deleted after commit and the queue row cleared on success. `jobs.Retention` retries queue entries older than one hour, so a crash between commit and object deletion never strands an object.
> *Enforcement:* `media.Service.Delete` + `jobs.Retention`.

- `07-data-model.md` System Tables gains: `cms_media_deletions` | `object_key` (PK), `created_at` | Deletion outbox (BR-MEDIA-5): enqueued in the delete transaction, cleared when the object delete succeeds, retried by `jobs.Retention`.
- 07 §Media Deletion is rewritten around the outbox (the "row deleted first, then object; orphan sweep is the backstop" claim is deleted); the destructive gate and FK-409 semantics stay.
- **BR-MEDIA-2 (amended):** the sweep sentence becomes "The orphan sweep deletes the storage object and row for `cms_media` rows stuck in `pending` after 24 hours." (row-driven wording, matching 07).
- 07 §Retention and 08 job telemetry gain duty 8: "Retries `cms_media_deletions` entries older than 1 hour: delete the object, then the row (BR-MEDIA-5)."
- `02-core-interfaces.md` `media.Service` gains `Delete(p Principal, mediaID UUID) error  // destructive-gated; 409 while referenced (BR-MEDIA-5)`; `04-api-layer.md` route-map media row gains "delete".

## 5. Argon2id Admission Control (D2-5, AR2-5)

**BR-AUTH-3 (amended, verbatim):**

> **BR-AUTH-3.** Password hashing uses Argon2id with per-hash salts (64 MiB memory, 3 iterations, parallelism 2). A global semaphore caps concurrent hash/verify operations at `min(4, NumCPU)`; work exceeding the cap waits up to 2 seconds, then fails with `429 rate_limited` — memory use from password hashing is bounded at ~256 MiB regardless of request volume.
> *Enforcement:* `auth.Password.Hash` / `auth.Password.Verify` behind a package-level semaphore.

`05-auth-security.md` §1 password-hashing bullet and §5 enumeration paragraph reference the semaphore (the uniform-Argon2id-work policy stands; the semaphore bounds its aggregate cost).

## 6. Public `contains` Removal (D2-6, AR2-6)

**BR-API-4 (amended, verbatim):**

> **BR-API-4.** Public-scope list queries accept `filter`/`sort` only on fields marked `indexed` or `unique`, with the operator set `eq, neq, lt, lte, gt, gte, in` — `contains` is admin- and trash-scope only in V1, because infix `ILIKE` cannot use B-tree indexes and would reopen the anonymous scan vector; violations return `422` naming the field or operator. Admin and trash scopes accept any schema field and the full operator set.
> *Enforcement:* `query.Builder` scope-aware field and operator validation.

- `04-api-layer.md` §Filtering states the public operator subset and keeps `contains → ILIKE` (escaped wildcards, text-only) for admin/trash.
- `02-core-interfaces.md` invariant 7 appends: "and the public operator set excludes `contains` (BR-API-4)."
- `03-dynamic-schema.md` (FTS section) and `11-roadmap.md` V2 scope note the re-enablement path: V2 may restore public `contains` behind a per-field `pg_trgm` GIN index (a bundled extension, N-9-compatible), alongside FTS.

## 7. Medium Sweep (D2-7)

### 7.1 Registration gate + end-user management (AR2-7)

**BR-AUTH-14 (new, verbatim):**

> **BR-AUTH-14.** End-user registration is enabled only when `CMS_END_USER_REGISTRATION=enabled` (default `disabled`); otherwise `POST /api/v1/auth/register` returns `404`. Admins manage end users at `/api/admin/end-users`: list, disable, re-enable, and revoke refresh-token families. Disabling sets `disabled_at`, revokes every refresh-token family, and the evaluator resolves the principal as `anonymous` until re-enabled; `login` and `refresh` for a disabled user return `401` with the uniform error shape.
> *Enforcement:* register handler gate + `httpapi/admin` end-user handlers + `access.Evaluator`.

- Env table (BUSINESS_RULES § Naming Constants) gains: `CMS_END_USER_REGISTRATION` | `disabled` | Enables end-user registration when set to `enabled` (BR-AUTH-14). The table is now **seventeen** variables: every "sixteen-variable" phrase in 09 (and anywhere else `grep -rn "sixteen" docs/` finds) updates.
- `07-data-model.md` `cms_end_users` row gains `disabled_at`.
- `04-api-layer.md` route map gains: `/api/admin/end-users` | session | End-user management: list, disable/enable, revoke refresh families (BR-AUTH-14).
- `06-admin-ui.md` Users & API keys screen: "includes an end-user tab: list, disable/enable, revoke sessions (BR-AUTH-14)."
- REQUIREMENTS gains **F-34 (V1)**: "Admins list, disable, re-enable end users and revoke their refresh-token families; end-user registration is env-gated and default-disabled (BR-AUTH-14)." — and **UAC-1.7**: "An admin disables an end user: the user's refresh replay returns 401 and subsequent API requests resolve as anonymous; re-enabling restores access. Separately, in a collection whose `create` grant sets `createStatus: "published"`, an end-user create is immediately publicly readable, while in a default collection the same user reads back their own draft that anonymous requests cannot see." F-11 appends "(registration env-gated per BR-AUTH-14)". `11-roadmap.md` V1 scope appends: end-user registration gate + admin end-user management; CORS contract (BR-API-6); media deletion outbox (BR-MEDIA-5); owner-draft visibility and `createStatus`.

### 7.2 Rate-limiter bound (AR2-8)

**BR-RUNTIME-4 (amended, verbatim):**

> **BR-RUNTIME-4.** Rate-limiting state lives in process memory and resets on restart. Buckets are held in a bounded LRU with a fixed entry cap; eviction forgets the bucket — memory is bounded at the cap regardless of client cardinality.
> *Enforcement:* `middleware.RateLimit` — in-memory LRU of token buckets keyed by email and client IP.

### 7.3 Idempotency completeness (AR2-9)

`04-api-layer.md` §Idempotent Creates gains (normative):

> The idempotency row inserts in the same transaction as the record — a crash cannot separate them. A concurrent request with the same key blocks on the unique index until the first transaction resolves: if it committed, the second request returns the original outcome; if it aborted, the second proceeds as a fresh create. The row stores a `request_hash` of the request body: presenting the same key with a different body returns `422 validation_failed`. The first create returns `201`; a replay returns the record's **current** representation with `200` — or `404` if the record has since been purged.

`07-data-model.md` `cms_idempotency_keys` row gains `request_hash`. `openapi.yaml`'s `Idempotency-Key` description notes the 200-replay and 422-on-mismatch.

### 7.4 DDL lock_timeout + pool constants (AR2-10)

`09-deployment.md` timeout table gains two rows: `Postgres lock_timeout — schema transactions | 5 s` and `pgx pool max connections | 10`. `03-dynamic-schema.md` Concurrency section gains: "The schema transaction runs `SET LOCAL lock_timeout = '5s'`: DDL that cannot acquire its table lock within 5 seconds aborts with `409 conflict` and a retry message instead of queueing all new traffic behind it; the shared 10-connection pool bounds how many requests can stack behind a stalled table, at the cost of coupling collections through the pool — schedule heavy changes off-peak."

### 7.5 Index naming (AR2-11)

`03-dynamic-schema.md` (whitelist section) and `07-data-model.md` (Indexes) gain: "Index names follow `ix_<table>_<field>`; when a name would exceed Postgres's 63-byte identifier limit, the join-table rule applies (each component truncates to its first 20 characters plus an 8-character hash of the full pair). RenameCollection and RenameField rename dependent indexes in the same transaction, keeping names deterministic — AddIndex's duplicate-detection ('duplicate index → no-op') depends on this."

### 7.6 NUMERIC precision (AR2-12)

`07-data-model.md` mapping becomes: "`number → NUMERIC(p,s)` when the field config declares precision/scale, bare `NUMERIC` otherwise". `03-dynamic-schema.md` matrix gains two rows: `number(p,s)` → `number` (bare) | Yes | widening to maximal; `number` (bare) → `number(p,s)` | No | narrowing. The definition-model sentence "numeric precision where applicable" stays and now has a defined meaning.

### 7.7 Anonymous read limits (AR2-13)

**BR-API-7 (new, verbatim):**

> **BR-API-7.** Anonymous public reads rate-limit at 300 requests per minute per client IP (`429 rate_limited` beyond). `?count=exact` requires an authenticated principal; anonymous use returns `422 validation_failed` naming the parameter.
> *Enforcement:* `middleware.RateLimit` anonymous-read bucket + `httpapi.ParsePagination`.

`04-api-layer.md` (envelope/pagination sections) and `05-auth-security.md` §5 reference it.

### 7.8 Cross-store restore consistency (AR2-14)

`09-deployment.md` §Backup gains: "The bucket is not point-in-time consistent with database PITR: a restore to time T resurrects `cms_media` rows whose objects were deleted after T (their delivery URLs 404) and leaves objects finalized after T unreferenced — acceptable residue, cleaned manually if it matters. After any restore, spot-check recent media rows against the bucket." The restore-drill runbook adds bucket reachability plus a spot-check of the most recent media rows to its checklist.

## 8. Low Sweep (D2-8)

| AR2 | Edit (target file(s)) |
|---|---|
| 15 | 05 §1/Recovery + 08 logging: setup/recovery token lines are "emitted regardless of `CMS_LOG_LEVEL`". CLAUDE.md hard rule 6 appends "(emitted regardless of log level)". |
| 16 | BR-AUTH-2 column list gains `csrf_hash`: "(`token_hash, user_id, csrf_hash, created_at, last_seen_at, ip, user_agent`)". |
| 17 | 01 architecture sketch middleware line gains `Recover` after `Logger` (matching 04's normative order). |
| 18 | **BR-RUNTIME-3 (amended):** "Startup executes in strict order: embedded migrations under advisory lock → instance lock (BR-RUNTIME-8) → schema cache load → HTTP listener." |
| 19 | 09 startup step 7 opens "(V2)" — the publisher and its catch-up tick exist from V2 (BR-LIFE-9). |
| 20 | **BR-AUTH-5 (amended)** appends: "The session cookie re-issues with a fresh `Max-Age` when 24 hours or more have passed since it was last set, never extending past the 30-day absolute bound." 05 §1 Expiry bullet mirrors it. |
| 21 | 09 §Runbooks: "**Master-secret rotation:** identical to loss recovery — set the new `CMS_MASTER_SECRET` and restart; the binary regenerates the keypair, outstanding ≤15-minute JWTs fail verification, and clients silently re-issue via refresh tokens. No dual-secret machinery exists or is needed." |
| 22 | 06 Security Posture: CSP gains `frame-ancestors 'none'`; admin responses add `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`. |
| 23 | 09 (Startup or Runbooks): the three advisory-lock keys are fixed constants — migration `0x636D7300`, schema `0x636D7301`, instance `0x636D7302` — and golang-cms assumes it is the only user of its database's advisory-lock keyspace (deploy it into a dedicated database). |
| 24 | 03 Definition Model: "An instance holds at most 500 collections; the 501st CreateCollection returns `422` — the same sanity-bound style as the 200-field cap." |
| 25 | 04 §Filtering: "`in` takes comma-separated values (`filter[status][in]=a,b`); `contains` (admin/trash scopes) applies to `text` fields only — any other type returns `422` naming the field." |
| 26 | `openapi.yaml`: `authRegister` success becomes `'201'`; the create-POST description notes idempotent replays return 200 with the current representation. |
| 27 | **BR-API-5 (amended)** appends: "A conditional GET presenting a matching `If-None-Match` returns `304 Not Modified`; the ETag is a strong hash of the response body." 09 §Runbooks gains a **cache-contract smoke check**: curl an anonymous public GET twice through the edge and confirm `s-maxage` + a cache hit; repeat with an `Authorization` header and confirm `no-store`; send `If-None-Match` with the received ETag and confirm `304`. |

## Identifier Delta Summary

- **New rules:** BR-API-6 (CORS), BR-API-7 (anonymous read limits), BR-AUTH-14 (registration gate + end-user management), BR-MEDIA-5 (deletion outbox).
- **Amended rules:** BR-API-2, BR-API-4, BR-API-5, BR-AUTH-2, BR-AUTH-3, BR-AUTH-5, BR-LIFE-2, BR-MEDIA-2, BR-RUNTIME-3, BR-RUNTIME-4, BR-RUNTIME-8.
- **New requirement / criterion:** F-34 (V1), UAC-1.7; F-11 and F-12 amended.
- **New table / columns:** `cms_media_deletions`; `cms_end_users.disabled_at`; `cms_idempotency_keys.request_hash`.
- **Env table:** +`CMS_END_USER_REGISTRATION` (default `disabled`) → **17 variables**; all "sixteen" phrasing updates.
- **Grant matrix:** +`createStatus` key (12 §1/§3/§4; §2 comments example updated).
- **Version stamps:** every edited doc bumps to `1.2 · 2026-07-11` (`12-access-rules.md` to `1.1`).

## Per-Document Edit Matrix

| File | Edits |
|---|---|
| `docs/BUSINESS_RULES.md` | New BR-API-6/7, BR-AUTH-14, BR-MEDIA-5; amendments listed above; env row + "seventeen"; header bump. |
| `docs/REQUIREMENTS.md` | F-11/F-12 amendments; new F-34, UAC-1.7. |
| `docs/architecture/01-system-overview.md` | Middleware sketch + `Recover`; lifecycle watchdog clause. |
| `docs/architecture/02-core-interfaces.md` | Invariant 3 (owner-draft) and 7 (operator set); `media.Service.Delete`. |
| `docs/architecture/03-dynamic-schema.md` | 500-collection cap; lock_timeout paragraph; index-naming rule; NUMERIC matrix rows; pg_trgm V2 note. |
| `docs/architecture/04-api-layer.md` | New CORS section; filtering operator subset + `in` encoding; idempotency paragraph; expansion sentence; media route "delete"; `/api/admin/end-users` row; BR-API-7 references. |
| `docs/architecture/05-auth-security.md` | Semaphore (§1, §5); BR-AUTH-14 + disabled semantics (§3 area); CORS threat-model line (§6); sliding cookie; token-emission note. |
| `docs/architecture/06-admin-ui.md` | End-user tab; CSP/frame-ancestors/nosniff/Referrer-Policy. |
| `docs/architecture/07-data-model.md` | `cms_media_deletions`; `disabled_at`; `request_hash`; Media Deletion rewrite; retention duty 8; index naming; NUMERIC mapping; createStatus contract line. |
| `docs/architecture/08-observability.md` | Retention duty 8; token-emission note. |
| `docs/architecture/09-deployment.md` | Startup step 3 retry + step 7 "(V2)"; "seventeen"; timeout-table rows; keepalive guidance; advisory-key constants; cross-store backup paragraph + drill additions; master-secret-rotation + cache-smoke runbooks. |
| `docs/architecture/11-roadmap.md` | V1 scope additions; V2 pg_trgm note. |
| `docs/architecture/12-access-rules.md` | `createStatus` row/algorithm/validation; comments example; owner-draft cross-reference; version → 1.1. |
| `docs/api/openapi.yaml` | register 201; idempotency description note. |
| `docs/reviews/architecture-review-round2-2026-07-11.md` | Append `## Resolution Status (2026-07-11)` table, AR2-1…AR2-27 → disposition (rule/section landed in). |
| `CLAUDE.md` | Hard rule 6 log-level clause. |
| `.claude/skills/*` (tracked) | Verify `auth-security-conventions` and `run-and-verify` against the above; fix any contradiction introduced by this pass (registration gate, operator set). |

Unlisted docs (`00-README.md`, `10-project-structure.md`) are expected unchanged; a task should verify no stale claims rather than edit them.

## Acceptance Criteria

1. `grep -rn "createStatus" docs/architecture/12-access-rules.md docs/BUSINESS_RULES.md docs/architecture/07-data-model.md` — present in all three; the 12 §2 comments example contains it.
2. `grep -rn "sixteen" docs/` → no matches (normative docs say seventeen).
3. `grep -n "BR-API-6\|BR-API-7\|BR-AUTH-14\|BR-MEDIA-5" docs/BUSINESS_RULES.md` → all four defined.
4. `grep -n "contains" docs/BUSINESS_RULES.md docs/architecture/04-api-layer.md` → public operator set excludes it; admin/trash retention stated.
5. `grep -rn "cms_media_deletions" docs/architecture/07-data-model.md docs/architecture/08-observability.md docs/BUSINESS_RULES.md` → present in all three; 07's Media Deletion section no longer claims the pending-sweep backstops the crash window.
6. `grep -rn "CMS_END_USER_REGISTRATION" docs/` → env table + BR-AUTH-14 + 04 route map.
7. `grep -n "LRU" docs/BUSINESS_RULES.md` → BR-RUNTIME-4 amended.
8. Round-2 review file ends with a Resolution Status table listing AR2-1…AR2-27, each with a disposition naming the landing rule/section.
9. Every edited markdown doc header reads `**Version:** 1.2 · **Last Updated:** 2026-07-11` (12-access-rules: 1.1); `openapi.yaml` keeps `info.version: 1.0.0` — the API surface itself is unchanged.
10. `make`-style sanity: no doc references `Vary: Origin`, dual master secrets, or an admin-principal `createStatus` — the rejected alternatives stay out.
