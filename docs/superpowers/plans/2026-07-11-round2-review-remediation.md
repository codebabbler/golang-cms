# Round-2 Review Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land the approved round-2 remediation design (D2-1…D2-8, resolving AR2-1…AR2-27) across the golang-cms documentation set — documentation only, no code.

**Architecture:** Task 1 amends the authority document (`docs/BUSINESS_RULES.md`) first; Tasks 2–13 propagate to the architecture docs, REQUIREMENTS, openapi.yaml, CLAUDE.md, and the two tracked skill files; Task 14 appends the Resolution Status table to the round-2 review and runs the acceptance sweep. Spec: `docs/superpowers/specs/2026-07-11-round2-review-remediation-design.md`.

**Tech Stack:** Markdown/YAML edits; grep for verification; git.

## Global Constraints

- **Documentation-only pass.** No Go/JS/SQL code, no files created or modified outside the ones each task names.
- **Verbatim texts.** Rule and section texts quoted in a task land character-for-character (backticks and punctuation included); only surrounding-sentence stitching may vary.
- **Authority chain:** `docs/BUSINESS_RULES.md` > `docs/architecture/*` > `.claude/skills/*`. On any conflict discovered mid-task, the higher document wins and the finding is reported, not silently resolved.
- **Version stamps:** every edited markdown doc header becomes `**Version:** 1.2 · **Last Updated:** 2026-07-11` — except `docs/architecture/12-access-rules.md`, which becomes `**Version:** 1.1 · **Last Updated:** 2026-07-11`. `docs/api/openapi.yaml` keeps `info.version: 1.0.0`.
- **Git: commits are authorized by the user (2026-07-11) with plain messages and NO Co-Authored-By trailer.** Commit exactly the files the task names. Never stage unrelated untracked files (`.claude/skills/system-design/`, `prompt.md`).
- **Env table is seventeen variables after Task 1.** No document may say "sixteen-variable" when this plan completes.
- **Rejected alternatives stay out:** no `Vary: Origin`, no dual/previous master secret, no `createStatus` for admin-kind principals, no bucket-LIST reconciliation sweep.
- The design points for records: `createStatus` applies to `end_user` and `api_key` creates only; owner-draft visibility never applies to anonymous requests nor to relation-expansion targets.

---

### Task 1: BUSINESS_RULES.md — new rules, amendments, 17-var env table

**Files:**
- Modify: `docs/BUSINESS_RULES.md`

**Interfaces:**
- Produces: BR-API-6, BR-API-7, BR-AUTH-14, BR-MEDIA-5 (new); amended BR-API-2/4/5, BR-AUTH-2/3/5, BR-LIFE-2, BR-MEDIA-2, BR-RUNTIME-3/4/8; env row `CMS_END_USER_REGISTRATION`. Every later task cites these identifiers exactly as written here.

- [ ] **Step 1: Pre-verify current text exists**

Run: `grep -n "schema cache load → HTTP listener\|in-memory token buckets\|held until exit\|Publishing copies the revision's data\|last_seen_at, ip, user_agent\|Argon2id with per-hash salts\.\|preceding 4 hours\.\|storage objects with no finalized\|grants draft access\.\|naming the field\. Admin and trash\|no-store\` without exception\.\|CMS_TRUSTED_PROXY_CIDRS" docs/BUSINESS_RULES.md`
Expected: matches on (approximately) lines 13, 16, 24, 49, 70, 72, 76, 116, 127, 131, 133, 160.

- [ ] **Step 2: Amend BR-RUNTIME-3**

Replace:
```
- **BR-RUNTIME-3.** Startup executes in strict order: embedded migrations under advisory lock → schema cache load → HTTP listener.
```
with:
```
- **BR-RUNTIME-3.** Startup executes in strict order: embedded migrations under advisory lock → instance lock (BR-RUNTIME-8) → schema cache load → HTTP listener.
```

- [ ] **Step 3: Amend BR-RUNTIME-4**

Replace:
```
- **BR-RUNTIME-4.** Rate-limiting state lives in process memory and resets on restart.
  *Enforcement:* `middleware.RateLimit` — in-memory token buckets keyed by email and client IP.
```
with:
```
- **BR-RUNTIME-4.** Rate-limiting state lives in process memory and resets on restart. Buckets are held in a bounded LRU with a fixed entry cap; eviction forgets the bucket — memory is bounded at the cap regardless of client cardinality.
  *Enforcement:* `middleware.RateLimit` — in-memory LRU of token buckets keyed by email and client IP.
```

- [ ] **Step 4: Amend BR-RUNTIME-8**

Replace:
```
- **BR-RUNTIME-8.** Exactly one process serves at a time: before opening the listener, the binary acquires a process-lifetime advisory lock (session-scoped, distinct from the migration and schema keys); a second process fails startup with a clear log line. *(Resolves EC-16.)*
  *Enforcement:* `app.Run` — `pg_advisory_lock` on the instance key, held until exit.
```
with:
```
- **BR-RUNTIME-8.** Exactly one process serves at a time: before opening the listener, the binary acquires a process-lifetime advisory lock (session-scoped, held on a dedicated connection with TCP keepalives, distinct from the migration and schema keys). Startup retries acquisition with backoff for up to 120 seconds — riding out a crashed predecessor's lingering session — then fails with a clear log line. If the lock connection drops at any point after acquisition, the process exits non-zero: serving without the lock is never permitted (fail closed, N-11). *(Resolves EC-16.)*
  *Enforcement:* `app.Run` — `pg_advisory_lock` on the instance key, dedicated connection, watchdog on connection loss.
```

- [ ] **Step 5: Amend BR-LIFE-2**

Replace:
```
- **BR-LIFE-2.** The live row holds the current published content for published records; drafts newer than the published version exist only in `cms_revisions`. Publishing copies the revision's data into the live row.
```
with:
```
- **BR-LIFE-2.** The live row holds the current published content for published records; drafts newer than the published version exist only in `cms_revisions`. Publishing copies the revision's data into the live row. Records created with `createStatus: "published"` (`docs/architecture/12-access-rules.md`) start published: the create transaction writes the live row as published and marks the first revision published.
```

- [ ] **Step 6: Amend BR-AUTH-2 (add `csrf_hash`)**

Replace `(`token_hash, user_id, created_at, last_seen_at, ip, user_agent`)` with `(`token_hash, user_id, csrf_hash, created_at, last_seen_at, ip, user_agent`)` inside BR-AUTH-2.

- [ ] **Step 7: Amend BR-AUTH-3**

Replace:
```
- **BR-AUTH-3.** Password hashing uses Argon2id with per-hash salts.
  *Enforcement:* `auth.Password.Hash` / `auth.Password.Verify`.
```
with:
```
- **BR-AUTH-3.** Password hashing uses Argon2id with per-hash salts (64 MiB memory, 3 iterations, parallelism 2). A global semaphore caps concurrent hash/verify operations at `min(4, NumCPU)`; work exceeding the cap waits up to 2 seconds, then fails with `429 rate_limited` — memory use from password hashing is bounded at ~256 MiB regardless of request volume.
  *Enforcement:* `auth.Password.Hash` / `auth.Password.Verify` behind a package-level semaphore.
```

- [ ] **Step 8: Amend BR-AUTH-5 (sliding cookie)**

Replace:
```
- **BR-AUTH-5.** Sessions expire after 7 idle days and 30 absolute days. Destructive operations require re-authentication within the preceding 4 hours.
```
with:
```
- **BR-AUTH-5.** Sessions expire after 7 idle days and 30 absolute days. Destructive operations require re-authentication within the preceding 4 hours. The session cookie re-issues with a fresh `Max-Age` when 24 hours or more have passed since it was last set, never extending past the 30-day absolute bound.
```

- [ ] **Step 9: Add BR-AUTH-14 (new, immediately after BR-AUTH-13's enforcement line)**

```
- **BR-AUTH-14.** End-user registration is enabled only when `CMS_END_USER_REGISTRATION=enabled` (default `disabled`); otherwise `POST /api/v1/auth/register` returns `404`. Admins manage end users at `/api/admin/end-users`: list, disable, re-enable, and revoke refresh-token families. Disabling sets `disabled_at`, revokes every refresh-token family, and the evaluator resolves the principal as `anonymous` until re-enabled; `login` and `refresh` for a disabled user return `401` with the uniform error shape.
  *Enforcement:* register handler gate + `httpapi/admin` end-user handlers + `access.Evaluator`.
```

- [ ] **Step 10: Amend BR-MEDIA-2 (sweep wording)**

Replace `The orphan sweep deletes storage objects with no finalized media record after 24 hours.` with `The orphan sweep deletes the storage object and row for `cms_media` rows stuck in `pending` after 24 hours.` inside BR-MEDIA-2.

- [ ] **Step 11: Add BR-MEDIA-5 (new, immediately after BR-MEDIA-4's enforcement line)**

```
- **BR-MEDIA-5.** Media deletion runs one transaction that deletes the `cms_media` row — FK RESTRICT produces the `409` while any record references it — and inserts the `object_key` into `cms_media_deletions`; the storage object is deleted after commit and the queue row cleared on success. `jobs.Retention` retries queue entries older than one hour, so a crash between commit and object deletion never strands an object.
  *Enforcement:* `media.Service.Delete` + `jobs.Retention`.
```

- [ ] **Step 12: Amend BR-API-2 (owner-draft visibility)**

Replace:
```
- **BR-API-2.** Public and API-key reads return only published, non-trashed records unless the key scope explicitly grants draft access.
```
with:
```
- **BR-API-2.** Public and API-key reads return only published, non-trashed records unless the key scope explicitly grants draft access or the record was created by the requesting principal: authenticated public principals (end users and API keys) always see records they created, including drafts (owner-draft visibility). Anonymous reads remain published-only without exception.
```

- [ ] **Step 13: Amend BR-API-4 (operator set)**

Replace:
```
- **BR-API-4.** Public-scope list queries accept `filter`/`sort` only on fields marked `indexed` or `unique`; violations return `422` naming the field. Admin and trash scopes accept any schema field.
  *Enforcement:* `query.Builder` scope-aware field validation.
```
with:
```
- **BR-API-4.** Public-scope list queries accept `filter`/`sort` only on fields marked `indexed` or `unique`, with the operator set `eq, neq, lt, lte, gt, gte, in` — `contains` is admin- and trash-scope only in V1, because infix `ILIKE` cannot use B-tree indexes and would reopen the anonymous scan vector; violations return `422` naming the field or operator. Admin and trash scopes accept any schema field and the full operator set.
  *Enforcement:* `query.Builder` scope-aware field and operator validation.
```

- [ ] **Step 14: Amend BR-API-5 (304 semantics)**

Replace:
```
- **BR-API-5.** Anonymous public reads carry `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` and a strong `ETag`; any request bearing `Authorization` or a cookie receives `Cache-Control: no-store` without exception.
```
with:
```
- **BR-API-5.** Anonymous public reads carry `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` and a strong `ETag`; any request bearing `Authorization` or a cookie receives `Cache-Control: no-store` without exception. A conditional GET presenting a matching `If-None-Match` returns `304 Not Modified`; the ETag is a strong hash of the response body.
```

- [ ] **Step 15: Add BR-API-6 and BR-API-7 (new, immediately after BR-API-5's enforcement line, in this order)**

```
- **BR-API-6.** Every `/api/v1` response carries `Access-Control-Allow-Origin: *`. Preflight `OPTIONS` requests are handled before authentication and rate limiting and answered with `Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key`, `Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS`, and `Access-Control-Max-Age: 86400`; non-preflight responses expose `X-Request-ID` and `ETag` via `Access-Control-Expose-Headers`. `/api/admin/*` and the SPA emit no CORS headers — the admin surface is same-origin only (cookie + CSRF). The wildcard is safe because bearer tokens are not ambient credentials: CORS never causes a browser to attach a victim's JWT, and cookies are never accepted on `/api/v1`.
  *Enforcement:* `middleware.CORS` mounted on the `/api/v1` subtree only.
- **BR-API-7.** Anonymous public reads rate-limit at 300 requests per minute per client IP (`429 rate_limited` beyond). `?count=exact` requires an authenticated principal; anonymous use returns `422 validation_failed` naming the parameter.
  *Enforcement:* `middleware.RateLimit` anonymous-read bucket + `httpapi.ParsePagination`.
```

- [ ] **Step 16: Env table row (after the `CMS_TRUSTED_PROXY_CIDRS` row)**

```
| `CMS_END_USER_REGISTRATION` | `disabled` | Enables end-user registration when set to `enabled` (BR-AUTH-14). |
```

- [ ] **Step 17: Bump header**

Replace `**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal` with `**Version:** 1.2 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal`.

- [ ] **Step 18: Post-verify**

Run: `grep -c "BR-API-6\|BR-API-7\|BR-AUTH-14\|BR-MEDIA-5" docs/BUSINESS_RULES.md` → ≥ 4. Then `grep -n "LRU\|csrf_hash\|createStatus\|owner-draft\|If-None-Match\|CMS_END_USER_REGISTRATION\|120 seconds\|min(4, NumCPU)\|instance lock (BR-RUNTIME-8) → schema cache" docs/BUSINESS_RULES.md` → all present. `grep -n "Version: 1.1\|held until exit\|storage objects with no finalized" docs/BUSINESS_RULES.md` → no matches.

- [ ] **Step 19: Commit**

```bash
git add docs/BUSINESS_RULES.md
git commit -m "docs: round-2 BUSINESS_RULES — BR-API-6/7, BR-AUTH-14, BR-MEDIA-5, 11 amendments, 17-var env table"
```

---

### Task 2: 12-access-rules.md — createStatus + owner-draft visibility

**Files:**
- Modify: `docs/architecture/12-access-rules.md`

**Interfaces:**
- Consumes: BR-API-2 (owner-draft), BR-LIFE-2 (createStatus start-published), from Task 1 — cite them by ID, do not restate their full text.
- Produces: the `createStatus` grant key that Tasks 3, 5, 7 cite.

- [ ] **Step 1: Pre-verify**

Run: `grep -n "anonymous\` | bool\|\"minRole\": \"contributor\" }\|unknown grant key\|Unpublish evaluates\|reads additionally scoped by ScopePublic\|reads published-only unless\|Version:\*\* 1.0" docs/architecture/12-access-rules.md`
Expected: all present.

- [ ] **Step 2: §1 grant-object table — add row after the `anonymous` row**

```
| `createStatus` | `"draft"` \| `"published"` | Valid on `create` only. The initial `status` of records created by `end_user` or `api_key` principals; default `draft`. Never applies to admin-kind principals — their creates always start as `draft` and go live only through the `publish` action. |
```

- [ ] **Step 3: §2 comments example — auto-publish comments**

Replace:
```
  "create":  { "endUsers": "all", "minRole": "contributor" },
```
with:
```
  "create":  { "endUsers": "all", "minRole": "contributor", "createStatus": "published" },  // comments go live on creation
```

- [ ] **Step 4: §3 — owner-draft in steps 3 and 5**

Replace:
```
3. `end_user`: per `endUsers` (`own` ⇒ `ownerOnly`); reads additionally scoped by ScopePublic (BR-API-2).
```
with:
```
3. `end_user`: per `endUsers` (`own` ⇒ `ownerOnly`); reads additionally scoped by ScopePublic, which includes the principal's own records regardless of status (owner-draft visibility, BR-API-2).
```
Replace:
```
5. `api_key`: allowed iff the key's scopes contain `(collection, action)`; reads published-only unless `draftAccess` (§6). Revoked keys resolve to `anonymous`.
```
with:
```
5. `api_key`: allowed iff the key's scopes contain `(collection, action)`; reads published-only unless `draftAccess` (§6) — records the key itself created are always visible (owner-draft visibility, BR-API-2). Revoked keys resolve to `anonymous`.
```

- [ ] **Step 5: §3 — createStatus semantics paragraph (new, immediately after the paragraph ending "Unpublish evaluates as the `publish` action — BR-LIFE-3 governs both.")**

```
For `action = create` by an `end_user` or `api_key` principal, the created record's initial `status` is the grant's `createStatus` (default `draft`). `createStatus: "published"` is initial state, not a `publish` action: BR-LIFE-3's editor floor governs the `publish` action only, and because `createStatus` never applies to admin-kind principals, no staff role can use it to bypass that floor. A `createStatus: "published"` create writes the live row as published and its first revision with `published = true` in the same transaction (BR-LIFE-2), and emits `content.record.create` with `createStatus` in the audit detail.
```

- [ ] **Step 6: §4 validation — extend the closed key set and add two bullets**

Replace:
```
- an unknown grant key (outside `minRole`, `minRoleOwn`, `endUsers`, `anonymous`),
```
with:
```
- an unknown grant key (outside `minRole`, `minRoleOwn`, `endUsers`, `anonymous`, `createStatus`),
```
Then add two bullets after the `anonymous` bullet ("`anonymous` present on any action other than `read`, or"), preserving list punctuation:
```
- `createStatus` present on any action other than `create`,
- a non-enum `createStatus` value (outside `draft`, `published`), or
```
(The pre-existing final bullet "a non-boolean `anonymous` value or a non-enum `endUsers` value." keeps the closing period; adjust the "or" so exactly the second-to-last bullet carries it.)

- [ ] **Step 7: §6 — createStatus reaches API-key creates**

Add after the paragraph ending "a second gate would add confusion, not security.":
```
One matrix key does reach API-key writes: a create through a key takes the collection's `createStatus` (§1) as its initial status — the key's scopes decide *whether* the create is allowed; the matrix decides what status it lands in.
```

- [ ] **Step 8: "Rules Resolved Here" — add BR-API-2 line**

Add after the BR-RBAC-7 line:
```
- **BR-API-2** — §3 steps 3 and 5 (owner-draft visibility on ScopePublic reads).
```

- [ ] **Step 9: Bump header to 1.1**

Replace `**Version:** 1.0 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal` with `**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal`.

- [ ] **Step 10: Post-verify**

Run: `grep -c "createStatus" docs/architecture/12-access-rules.md` → ≥ 7. `grep -n "owner-draft" docs/architecture/12-access-rules.md` → ≥ 3 matches.

- [ ] **Step 11: Commit**

```bash
git add docs/architecture/12-access-rules.md
git commit -m "docs: access rules — createStatus grant key, owner-draft visibility (D2-1)"
```

---

### Task 3: REQUIREMENTS.md — F-11/F-12 amendments, F-34, UAC-1.7

**Files:**
- Modify: `docs/REQUIREMENTS.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "F-11 (V1)\|F-12 (V1)\|F-32 (V1)\|UAC-1.6" docs/REQUIREMENTS.md` → all present.

- [ ] **Step 2: Amend F-11**

Replace:
```
- **F-11 (V1).** The CMS acts as the user store for client applications: end users register and log in, receive a 15-minute RS256 JWT plus rotating refresh token, and lose the whole token family on refresh-token reuse. *(Resolves EC-8 via BR-AUTH-9.)*
```
with:
```
- **F-11 (V1).** The CMS acts as the user store for client applications: end users register and log in, receive a 15-minute RS256 JWT plus rotating refresh token, and lose the whole token family on refresh-token reuse; registration is env-gated and default-disabled (BR-AUTH-14). *(Resolves EC-8 via BR-AUTH-9.)*
```

- [ ] **Step 3: Amend F-12 (append sentence)**

Append to F-12, after "missing rules deny for the governed classes.":
```
 Collections may set `createStatus` to publish public-API creates immediately (`docs/architecture/12-access-rules.md` §1).
```

- [ ] **Step 4: Add F-34 (new, immediately after F-32)**

```
- **F-34 (V1).** Admins list, disable, re-enable end users and revoke their refresh-token families; end-user registration is env-gated and default-disabled (BR-AUTH-14).
```

- [ ] **Step 5: Add UAC-1.7 (new, immediately after UAC-1.6)**

```
- **UAC-1.7.** An admin disables an end user: the user's refresh replay returns 401 and subsequent API requests resolve as anonymous; re-enabling restores access. Separately, in a collection whose `create` grant sets `createStatus: "published"`, an end-user create is immediately publicly readable, while in a default collection the same user reads back their own draft that anonymous requests cannot see.
```

- [ ] **Step 6: Bump header to 1.2; Post-verify**

Run: `grep -n "F-34\|UAC-1.7\|createStatus\|BR-AUTH-14" docs/REQUIREMENTS.md` → all present.

- [ ] **Step 7: Commit**

```bash
git add docs/REQUIREMENTS.md
git commit -m "docs: REQUIREMENTS — F-34, UAC-1.7, F-11/F-12 round-2 amendments"
```

---

### Task 4: 02-core-interfaces.md — builder invariants, media.Service.Delete

**Files:**
- Modify: `docs/architecture/02-core-interfaces.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "ScopePublic\` appends \`status = 'published'\|accepts filter/sort only on indexed\|Finalize(p Principal" docs/architecture/02-core-interfaces.md` → all present.

- [ ] **Step 2: Amend invariant 3**

Replace:
```
3. `ScopePublic` appends `status = 'published'` (BR-API-2).
```
with:
```
3. `ScopePublic` appends `status = 'published'` (BR-API-2), relaxed to `(status = 'published' OR created_by = <principal>)` when the Decision carries an authenticated public principal (owner-draft visibility); anonymous requests get the strict form.
```

- [ ] **Step 3: Amend invariant 7**

Replace:
```
7. `ScopePublic` accepts filter/sort only on indexed or unique fields (BR-API-4); cursor pagination preserves the `id` tiebreaker and limit clamps.
```
with:
```
7. `ScopePublic` accepts filter/sort only on indexed or unique fields, and its operator set excludes `contains` (BR-API-4); cursor pagination preserves the `id` tiebreaker and limit clamps.
```

- [ ] **Step 4: Add `Delete` to media.Service**

Replace:
```
Presign(p Principal, req UploadRequest) (PresignedUpload, error)  // ≤15 min, size-capped
Finalize(p Principal, mediaID UUID) (Media, error)                 // verifies object existence
```
with:
```
Presign(p Principal, req UploadRequest) (PresignedUpload, error)  // ≤15 min, size-capped
Finalize(p Principal, mediaID UUID) (Media, error)                 // verifies object existence
Delete(p Principal, mediaID UUID) error                            // destructive-gated; 409 while referenced (BR-MEDIA-5)
```
And replace the guarantees sentence:
```
Guarantees: no method accepts file bytes (BR-MEDIA-1); `Finalize` flips `pending → finalized` only after a storage HEAD confirms the object within the declared size (BR-MEDIA-2).
```
with:
```
Guarantees: no method accepts file bytes (BR-MEDIA-1); `Finalize` flips `pending → finalized` only after a storage HEAD confirms the object within the declared size (BR-MEDIA-2); `Delete` removes the row and enqueues the object key in one transaction, with object deletion after commit (BR-MEDIA-5).
```

- [ ] **Step 5: Bump header to 1.2; Post-verify**

Run: `grep -n "owner-draft visibility\|excludes \`contains\`\|BR-MEDIA-5" docs/architecture/02-core-interfaces.md` → all present.

- [ ] **Step 6: Commit**

```bash
git add docs/architecture/02-core-interfaces.md
git commit -m "docs: core interfaces — owner-draft invariant, public operator set, media.Service.Delete"
```

---

### Task 5: 04-api-layer.md — CORS section, filtering grammar, idempotency, routes

**Files:**
- Modify: `docs/architecture/04-api-layer.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "presign\`, \`finalize\`, listing\|End-user \`register\`\|rate limiting precedes authentication\|public consumers default to no count\|op ∈ eq, neq\|## Idempotent Creates\|serialize as \`null\`\|## Caching" docs/architecture/04-api-layer.md` → all present.

- [ ] **Step 2: Route map edits**

Replace `| `/api/admin/media` | session | `presign`, `finalize`, listing. |` with `| `/api/admin/media` | session | `presign`, `finalize`, listing, delete (BR-MEDIA-5). |`

Add a new row immediately after the media row:
```
| `/api/admin/end-users` | session | End-user management: list, disable/enable, revoke refresh families (BR-AUTH-14). |
```
Replace `End-user `register`, `login`,` with `End-user `register` (env-gated, BR-AUTH-14), `login`,` in the `/api/v1/auth/*` row.

- [ ] **Step 3: Middleware-order commentary — anonymous read limit**

Replace `rate limiting precedes authentication so credential stuffing burns the limiter, not Argon2id time (BR-AUTH-6);` with `rate limiting precedes authentication so credential stuffing burns the limiter, not Argon2id time (BR-AUTH-6); anonymous public reads limit at 300 requests/min per client IP (BR-API-7);`

- [ ] **Step 4: Envelope section — count=exact restriction**

Append to the paragraph ending "and the admin UI requests it where its tables need totals.":
```
 `?count=exact` requires an authenticated principal — anonymous use returns `422 validation_failed` naming the parameter (BR-API-7).
```

- [ ] **Step 5: Rewrite the Filtering and Sorting paragraph**

Replace the whole paragraph under `## Filtering and Sorting (BR-API-4)` with:
```
`?filter[<field>][<op>]=<value>` with `op ∈ eq, neq, lt, lte, gt, gte, in, contains`; `in` takes comma-separated values (`filter[status][in]=a,b`); `?sort=<field>` / `?sort=-<field>`. Fields must exist in the schema snapshot and survive `Decision.FieldRules` visibility; violations return `422` naming the field. `ScopePublic` queries accept `filter`/`sort` only on fields marked `indexed` or `unique`, with the operator subset `eq, neq, lt, lte, gt, gte, in` — `contains` is admin- and trash-scope only in V1 (BR-API-4): infix `ILIKE` cannot use B-tree indexes, and V2 may restore public `contains` behind a per-field `pg_trgm` GIN index. A public request naming any other field or operator returns `422 validation_failed` naming the offender. `ScopeAdmin` and `ScopeTrash` accept any schema field and the full operator set, still subject to `Decision.FieldRules` visibility and the query's `statement_timeout`. All composition happens inside `query.Builder` — operators are a closed set, and `contains` maps to `ILIKE` with escaped wildcards on `text` fields only; `contains` on any non-`text` field returns `422` naming the field.
```

- [ ] **Step 6: Caching section — owner-draft note**

Append to the `## Caching (BR-API-5)` paragraph:
```
 Owner-draft-widened responses (BR-API-2) are always credentialed and therefore already `no-store`; the anonymous cacheable class is unaffected.
```

- [ ] **Step 7: New CORS section (immediately after the Caching section)**

```
## CORS (BR-API-6)

Every `/api/v1` response carries `Access-Control-Allow-Origin: *`. Preflight `OPTIONS` requests are handled before authentication and rate limiting and answered with `Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key`, `Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS`, and `Access-Control-Max-Age: 86400`; non-preflight responses expose `X-Request-ID` and `ETag` via `Access-Control-Expose-Headers`. `/api/admin/*` and the SPA emit no CORS headers — the admin surface is same-origin only (cookie + CSRF, BR-AUTH-4). The wildcard is safe because bearer tokens are not ambient credentials: CORS never causes a browser to attach a victim's JWT, and cookies are never accepted on `/api/v1`. Because `Access-Control-Allow-Origin` is the constant `*`, the cached anonymous responses of BR-API-5 need no `Vary: Origin`.
```

- [ ] **Step 8: Idempotent Creates — completeness paragraph (append to the section)**

```
The idempotency row inserts in the same transaction as the record — a crash cannot separate them. A concurrent request with the same key blocks on the unique index until the first transaction resolves: if it committed, the second request returns the original outcome; if it aborted, the second proceeds as a fresh create. The row stores a `request_hash` of the request body: presenting the same key with a different body returns `422 validation_failed`. The first create returns `201`; a replay returns the record's **current** representation with `200` — or `404` if the record has since been purged.
```

- [ ] **Step 9: Relation Expansion — owner-draft boundary (append to the section's first paragraph)**

```
 Expansion targets resolve strictly published-only even for their owners — owner-draft visibility (BR-API-2) applies to the requested records, never to expanded targets.
```

- [ ] **Step 10: Bump header to 1.2; Post-verify**

Run: `grep -n "## CORS (BR-API-6)\|request_hash\|end-users\|operator subset\|BR-API-7" docs/architecture/04-api-layer.md` → all present. `grep -n "Vary: Origin" docs/architecture/04-api-layer.md` → only inside "need no `Vary: Origin`".

- [ ] **Step 11: Commit**

```bash
git add docs/architecture/04-api-layer.md
git commit -m "docs: API layer — CORS section, public operator subset, idempotency completeness, end-user routes"
```

---

### Task 6: 03-dynamic-schema.md — collection cap, lock_timeout, index naming, NUMERIC, pg_trgm

**Files:**
- Modify: `docs/architecture/03-dynamic-schema.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "at most 200 user fields\|AddForeignKey/DropForeignKey apply\|number(p,s)\` → \`number(p′,s′)\|single-tenant admin population\|both strategies guarantee" docs/architecture/03-dynamic-schema.md` → all present.

- [ ] **Step 2: Collection cap (new bullet after the 200-field bullet in Definition Model)**

```
- An instance holds at most 500 collections; the 501st CreateCollection returns `422` — the same sanity-bound style as the 200-field cap.
```

- [ ] **Step 3: Index-naming paragraph (new, immediately after the media-FK paragraph ending "AddForeignKey/DropForeignKey apply to `relation` fields only.")**

```
Index names follow `ix_<table>_<field>`; when a name would exceed Postgres's 63-byte identifier limit, the join-table rule of `07-data-model.md` applies (each component truncates to its first 20 characters plus an 8-character hash of the full pair). RenameCollection and RenameField rename dependent indexes in the same transaction, keeping names deterministic — AddIndex's duplicate detection ("duplicate index → no-op") depends on this.
```

- [ ] **Step 4: Safe-conversion matrix — two NUMERIC rows (immediately after the `number(p,s)` widening row)**

```
| `number(p,s)` → `number` (bare) | Yes | widening to maximal |
| `number` (bare) → `number(p,s)` | No | narrowing |
```

- [ ] **Step 5: lock_timeout paragraph (new, at the end of the Concurrency and Atomicity section, after mechanism 3's paragraph)**

```
The schema transaction runs `SET LOCAL lock_timeout = '5s'`: DDL that cannot acquire its table lock within 5 seconds aborts with `409 conflict` and a retry message instead of queueing all new traffic behind it. The shared 10-connection pool (`09-deployment.md`) bounds how many requests can stack behind a stalled table, at the cost of coupling collections through the pool — schedule heavy changes off-peak.
```

- [ ] **Step 6: pg_trgm note (append to the FTS Interaction section, after the V2 Alternative paragraph)**

```
V2 may also restore public `contains` filtering (removed from `ScopePublic` in V1 — BR-API-4) behind a per-field `pg_trgm` GIN index; `pg_trgm` is a bundled extension, so N-9's restorability guarantee holds.
```

- [ ] **Step 7: Bump header to 1.2; Post-verify**

Run: `grep -n "500 collections\|lock_timeout\|ix_<table>_<field>\|widening to maximal\|pg_trgm" docs/architecture/03-dynamic-schema.md` → all present.

- [ ] **Step 8: Commit**

```bash
git add docs/architecture/03-dynamic-schema.md
git commit -m "docs: schema engine — collection cap, DDL lock_timeout, index naming, NUMERIC precision, pg_trgm path"
```

---

### Task 7: 07-data-model.md — tables/columns, media-deletion rewrite, retention duty 8

**Files:**
- Modify: `docs/architecture/07-data-model.md`

**Interfaces:**
- Consumes: BR-MEDIA-5, BR-AUTH-14 (Task 1); `createStatus` (Task 2).

- [ ] **Step 1: Pre-verify**

Run: `grep -n "cms_end_users\` | \`id\`\|cms_idempotency_keys\` | \`key_hash\|number→NUMERIC\|orphan sweep (BR-MEDIA-2) is the backstop\|Sweeps \`cms_media\` rows\|partial unique index\*\* \`WHERE deleted_at IS NULL\`" docs/architecture/07-data-model.md` → all present.

- [ ] **Step 2: `cms_end_users` row — add `disabled_at`**

Replace:
```
| `cms_end_users` | `id`, `email` (unique), `password_hash`, `created_at`, `updated_at` | The JWT user store (F-11); separate from `cms_users` because the threat models differ. |
```
with:
```
| `cms_end_users` | `id`, `email` (unique), `password_hash`, `disabled_at`, `created_at`, `updated_at` | The JWT user store (F-11); separate from `cms_users` because the threat models differ. Non-NULL `disabled_at` = disabled: refresh families revoked, principal resolves as `anonymous` (BR-AUTH-14). |
```

- [ ] **Step 3: `cms_idempotency_keys` row — add `request_hash`**

Replace:
```
| `cms_idempotency_keys` | `key_hash`, `principal_id`, `record_id`, `created_at` | Unique on `(key_hash, principal_id)`; backs `Idempotency-Key` on public/API-key creates (`04-api-layer.md`); purged by `jobs.Retention` 24 h after creation. |
```
with:
```
| `cms_idempotency_keys` | `key_hash`, `principal_id`, `record_id`, `request_hash`, `created_at` | Unique on `(key_hash, principal_id)`; `request_hash` detects same-key/different-body replays (`422`); backs `Idempotency-Key` on public/API-key creates (`04-api-layer.md`); purged by `jobs.Retention` 24 h after creation. |
```

- [ ] **Step 4: Add `cms_media_deletions` row (immediately after the `cms_idempotency_keys` row)**

```
| `cms_media_deletions` | `object_key` (PK), `created_at` | Deletion outbox (BR-MEDIA-5): enqueued in the delete transaction, cleared when the object delete succeeds, retried by `jobs.Retention` after 1 h. |
```

- [ ] **Step 5: NUMERIC mapping**

In the user-field storage sentence, replace `number→NUMERIC` with `number→NUMERIC(p,s)` when the field config declares precision/scale, bare `NUMERIC` otherwise` (keep the surrounding commas; the sentence remains one list).

- [ ] **Step 6: Index-naming sentence (append to the **Indexes.** paragraph)**

```
 Index names follow `ix_<table>_<field>` with the join-table truncation rule above when they would exceed the 63-byte limit; renames rename dependent indexes in the same transaction (`03-dynamic-schema.md`).
```

- [ ] **Step 7: Public-create bullet (new, in The Live-Table/Revisions Contract, immediately after the **Write.** bullet)**

```
- **Public-API create.** Records created through the public API take the collection's `createStatus` (`12-access-rules.md` §1): a `published` create writes the live row as published and the first revision with `published = true` in one transaction; the default is `draft`.
```

- [ ] **Step 8: Retention duty 8 (append to the numbered list in ## Retention)**

```
8. Retries `cms_media_deletions` entries older than 1 hour: delete the object, then the row (BR-MEDIA-5).
```

- [ ] **Step 9: Rewrite ## Media Deletion**

Replace the section's single paragraph with:
```
Deleting a `cms_media` row is destructive-gated the same way a collection drop is: re-authentication within the preceding 4-hour window (BR-AUTH-5) plus typed confirmation, per the BR-SCHEMA-7 pattern. While any `media` field on any record still references the row, its `ON DELETE RESTRICT` FK blocks the delete with `409 Conflict` naming the referencing record. On success, one transaction deletes the row and inserts the `object_key` into `cms_media_deletions`; the storage object is deleted after commit and the queue row cleared on success. `jobs.Retention` retries queue entries older than one hour, so a process death mid-delete never strands an object (BR-MEDIA-5).
```

- [ ] **Step 10: Bump header to 1.2; Post-verify**

Run: `grep -n "cms_media_deletions\|disabled_at\|request_hash\|Public-API create\|ix_<table>_<field>" docs/architecture/07-data-model.md` → all present. `grep -n "is the backstop for the crash window" docs/architecture/07-data-model.md` → no match.

- [ ] **Step 11: Commit**

```bash
git add docs/architecture/07-data-model.md
git commit -m "docs: data model — media deletion outbox, disabled_at, request_hash, createStatus contract, retention duty 8"
```

---

### Task 8: 05-auth-security.md — semaphore, registration gate, CORS threat line, sliding cookie

**Files:**
- Modify: `docs/architecture/05-auth-security.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "parameters pinned at 64 MiB\|7 days idle, 30 days absolute\|logs it exactly once at \`warn\`\|logs it once at \`warn\`\|which is the worse outcome\|Reuse detection" docs/architecture/05-auth-security.md` → all present.

- [ ] **Step 2: §1 password hashing — semaphore sentence (append to the bullet)**

```
 A global semaphore caps concurrent hash/verify operations at `min(4, NumCPU)`; work exceeding the cap waits up to 2 seconds, then fails with `429 rate_limited` — password hashing memory is bounded at ~256 MiB regardless of request volume (BR-AUTH-3).
```

- [ ] **Step 3: §1 Expiry — sliding cookie (append to the bullet)**

```
 The session cookie re-issues with a fresh `Max-Age` once 24 hours have passed since it was last set, never extending past the 30-day absolute bound (BR-AUTH-5).
```

- [ ] **Step 4: Token-emission notes**

In §1 First-Admin Bootstrap, replace `logs it exactly once at `warn`,` with `logs it exactly once at `warn` (emitted regardless of `CMS_LOG_LEVEL`),`.
In §Recovery Mode, replace `logs it once at `warn`, and enables `/recover`.` with `logs it once at `warn` (emitted regardless of `CMS_LOG_LEVEL`), and enables `/recover`.`

- [ ] **Step 5: §3 — registration gate & disable bullet (new, immediately after the **Reuse detection** bullet)**

```
- **Registration gate & disable (BR-AUTH-14):** registration is enabled only when `CMS_END_USER_REGISTRATION=enabled` (default `disabled`); otherwise `/api/v1/auth/register` returns 404. Admins manage end users at `/api/admin/end-users` (list, disable, re-enable, revoke refresh families). Disabling sets `disabled_at`, revokes every refresh-token family, and the evaluator resolves the principal as `anonymous`; `login`/`refresh` for a disabled user return the uniform `401`.
```

- [ ] **Step 6: §5 — BR-API-7 + semaphore cross-reference (append to the paragraph ending "which is the worse outcome.")**

```
 Anonymous public reads limit at 300 requests/min per client IP, and `?count=exact` is authenticated-only (BR-API-7). The Argon2id semaphore (§1, BR-AUTH-3) bounds the aggregate memory cost of the uniform-verification policy.
```

- [ ] **Step 7: §6 threat model — two new lines (append to the bullet list)**

```
- Cross-origin abuse of the admin surface is prevented by the absence of CORS headers on `/api/admin/*` plus cookie `SameSite=Lax` and CSRF (BR-API-6).
- Password-hashing memory exhaustion is bounded by the Argon2id admission semaphore (BR-AUTH-3).
```

- [ ] **Step 8: Bump header to 1.2; Post-verify**

Run: `grep -n "min(4, NumCPU)\|CMS_END_USER_REGISTRATION\|emitted regardless\|BR-API-7\|BR-API-6" docs/architecture/05-auth-security.md` → all present (emitted-regardless ×2).

- [ ] **Step 9: Commit**

```bash
git add docs/architecture/05-auth-security.md
git commit -m "docs: auth security — Argon2id semaphore, registration gate, CORS threat line, sliding cookie"
```

---

### Task 9: 09-deployment.md — startup retry, constants, runbooks, seventeen

**Files:**
- Modify: `docs/architecture/09-deployment.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "sixteen-variable" docs/architecture/09-deployment.md` → exactly 2 matches (Configuration section, Timeouts section). `grep -n "A second process fails startup immediately\|The publisher's first tick, immediately\|statement_timeout\` — schema transactions\|Restore drill\*\* (N-12\|Master-secret-loss recovery" docs/architecture/09-deployment.md` → all present.

- [ ] **Step 2: Replace both "sixteen-variable" occurrences with "seventeen-variable"**

- [ ] **Step 3: Rewrite startup step 3**

Replace:
```
3. Acquire the process-lifetime instance lock (BR-RUNTIME-8): a session-scoped `pg_advisory_lock` on a dedicated key, distinct from the migration and schema keys. A second process fails startup immediately with a clear log line — the migration lock is something a replacement process waits on, but the instance lock is not: stop-then-start remains the only supported upgrade path. *(Resolves EC-16.)*
```
with:
```
3. Acquire the process-lifetime instance lock (BR-RUNTIME-8): a session-scoped `pg_advisory_lock` on a dedicated key, held on a dedicated connection with TCP keepalives, distinct from the migration and schema keys. Acquisition retries with backoff for up to 120 seconds — riding out a crashed predecessor's lingering session — then fails startup with a clear log line. If the lock connection drops after acquisition, the process exits non-zero: serving without the lock is never permitted. Stop-then-start remains the only supported upgrade path. *(Resolves EC-16.)*
```

- [ ] **Step 4: Tag startup step 7 as V2**

Replace `7. The publisher's first tick, immediately after the listener opens, runs` with `7. (V2) The publisher's first tick, immediately after the listener opens, runs`.

- [ ] **Step 5: Advisory-key constants (new paragraph immediately after the startup-failure paragraph ending "rather than serving with partial state.")**

```
The three advisory-lock keys are fixed constants: migration `0x636D7300`, schema `0x636D7301`, instance `0x636D7302`. golang-cms assumes it is the only user of its database's advisory-lock keyspace — deploy it into a dedicated database.
```

- [ ] **Step 6: Timeout table — two new rows (after the schema `statement_timeout` row)**

```
| Postgres `lock_timeout` — schema transactions | 5 s |
| pgx pool max connections | 10 |
```

- [ ] **Step 7: Backup section — cross-store bullet (new, after the "Object storage:" bullet)**

```
- **Cross-store consistency:** the bucket is not point-in-time consistent with database PITR: a restore to time T resurrects `cms_media` rows whose objects were deleted after T (their delivery URLs 404) and leaves objects finalized after T unreferenced — acceptable residue, cleaned manually if it matters. After any restore, spot-check recent media rows against the bucket.
```

- [ ] **Step 8: Runbook additions**

Append to the **JWT key rotation** runbook paragraph (after the master-secret-loss sentence):
```
 **Master-secret rotation:** identical to loss recovery — set the new `CMS_MASTER_SECRET` and restart; the binary regenerates the keypair, outstanding ≤15-minute JWTs fail verification, and clients silently re-issue via refresh tokens. No dual-secret machinery exists or is needed.
```
Append to the **Restore drill** runbook paragraph:
```
 The drill also verifies bucket reachability and spot-checks the most recent media rows against their objects (cross-store consistency above).
```
Add two new runbook paragraphs before **Hot-DDL guidance**:
```
**Postgres keepalive tuning** (BR-RUNTIME-8): set `tcp_keepalives_idle=60`, `tcp_keepalives_interval=10`, `tcp_keepalives_count=3` (or `idle_session_timeout` on managed Postgres) so a dead process's session releases the instance lock within roughly two minutes — inside the startup retry window.

**Cache-contract smoke check** (BR-API-5, BR-API-6): curl an anonymous public GET twice through the edge and confirm the response carries `s-maxage=60` and the second request is served from cache; repeat with an `Authorization` header and confirm `Cache-Control: no-store`; send `If-None-Match` with the received ETag and confirm `304 Not Modified`. Run after any proxy configuration change.
```

- [ ] **Step 9: Bump header to 1.2; Post-verify**

Run: `grep -n "sixteen" docs/architecture/09-deployment.md` → no matches. `grep -n "120 seconds\|0x636D7300\|lock_timeout\|pool max connections\|Cross-store\|Master-secret rotation\|keepalive tuning\|smoke check\|(V2) The publisher" docs/architecture/09-deployment.md` → all present.

- [ ] **Step 10: Commit**

```bash
git add docs/architecture/09-deployment.md
git commit -m "docs: deployment — instance-lock retry+watchdog, advisory keys, pool/lock constants, cross-store backup, new runbooks"
```

---

### Task 10: 01-system-overview.md + 08-observability.md

**Files:**
- Modify: `docs/architecture/01-system-overview.md`
- Modify: `docs/architecture/08-observability.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "RateLimit → Auth (Session|APIKey|JWT)\|prevents accidental second running instances" docs/architecture/01-system-overview.md; grep -n "media orphans swept\|30-minute TTL" docs/architecture/08-observability.md` → all present.

- [ ] **Step 2: 01 — add Recover to the middleware sketch**

Replace:
```
 API consumer ────────► │  ├─ middleware: RequestID → Logger →        │
 End-user client ─────► │  │   RateLimit → Auth (Session|APIKey|JWT)  │
```
with:
```
 API consumer ────────► │  ├─ middleware: RequestID → Logger →        │
 End-user client ─────► │  │   Recover → RateLimit → Auth             │
                        │  │   (Session|APIKey|JWT)                   │
```
Pad the new lines with trailing spaces so the closing `│` stays column-aligned with the rest of the box.

- [ ] **Step 3: 01 — watchdog clause + V2 tag in Lifecycle Summary**

Replace:
```
the publisher's first tick runs the missed-schedule catch-up (BR-RUNTIME-3, BR-LIFE-9). *(Resolves EC-16.)* The instance lock prevents accidental second running instances.
```
with:
```
the publisher's first tick (V2) runs the missed-schedule catch-up (BR-RUNTIME-3, BR-LIFE-9). *(Resolves EC-16.)* The instance lock prevents accidental second running instances; it lives on a dedicated watched connection, and losing it terminates the process (BR-RUNTIME-8).
```

- [ ] **Step 4: 08 — retention duty + token-emission note**

Replace `revisions pruned, idempotency rows > 24 h purged, media orphans swept.` with `revisions pruned, idempotency rows > 24 h purged, media orphans swept, media deletion-queue entries older than 1 h retried (BR-MEDIA-5).`

Replace `logged once at `warn` with a 30-minute TTL.` with `logged once at `warn` with a 30-minute TTL and emitted regardless of `CMS_LOG_LEVEL`.`

- [ ] **Step 5: Bump both headers to 1.2; Post-verify**

Run: `grep -n "Recover" docs/architecture/01-system-overview.md; grep -n "BR-MEDIA-5\|emitted regardless" docs/architecture/08-observability.md` → all present.

- [ ] **Step 6: Commit**

```bash
git add docs/architecture/01-system-overview.md docs/architecture/08-observability.md
git commit -m "docs: overview + observability — Recover in sketch, lock watchdog, retention duty 8, token emission"
```

---

### Task 11: 06-admin-ui.md + 11-roadmap.md

**Files:**
- Modify: `docs/architecture/06-admin-ui.md`
- Modify: `docs/architecture/11-roadmap.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "passwordReset\` capability toggle\|Strict CSP" docs/architecture/06-admin-ui.md; grep -n "idempotent record creates\|purge-on-publish lengthening" docs/architecture/11-roadmap.md` → all present.

- [ ] **Step 2: 06 — end-user tab (append to the Users & API keys screen cell, before the closing `|`)**

```
 Includes an end-user tab: list, disable/enable, revoke sessions (BR-AUTH-14).
```

- [ ] **Step 3: 06 — CSP and companion headers**

Replace:
```
- Strict CSP: `default-src 'self'; img-src 'self' <media domain>; media-src 'self' <media domain>`; no inline scripts.
```
with:
```
- Strict CSP: `default-src 'self'; img-src 'self' <media domain>; media-src 'self' <media domain>; frame-ancestors 'none'`; no inline scripts. Admin responses also carry `X-Content-Type-Options: nosniff` and `Referrer-Policy: strict-origin-when-cross-origin`.
```
(The rest of the bullet — media-domain configuration and Tiptap bundling — is unchanged.)

- [ ] **Step 4: 11 — V1 scope additions**

In the **Version 1 (MVP)** bullet, replace `idempotent record creates via idempotency keys.` with `idempotent record creates via idempotency keys; CORS contract for the public API (BR-API-6); end-user registration gate and admin end-user management (BR-AUTH-14, F-34); media deletion outbox (BR-MEDIA-5); owner-draft visibility and per-collection `createStatus` (BR-API-2, `12-access-rules.md`).`

- [ ] **Step 5: 11 — V2 scope addition**

In the **Version 2** bullet, replace `purge-on-publish lengthening cache TTLs.` with `purge-on-publish lengthening cache TTLs; public `contains` filtering restored behind per-field `pg_trgm` GIN indexes (BR-API-4).`

- [ ] **Step 6: Bump both headers to 1.2; Post-verify**

Run: `grep -n "end-user tab\|frame-ancestors\|nosniff" docs/architecture/06-admin-ui.md; grep -n "BR-AUTH-14\|pg_trgm\|BR-MEDIA-5" docs/architecture/11-roadmap.md` → all present.

- [ ] **Step 7: Commit**

```bash
git add docs/architecture/06-admin-ui.md docs/architecture/11-roadmap.md
git commit -m "docs: admin UI + roadmap — end-user tab, CSP hardening, round-2 V1/V2 scope additions"
```

---

### Task 12: openapi.yaml — register 201, idempotency replay note

**Files:**
- Modify: `docs/api/openapi.yaml`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "authRegister" docs/api/openapi.yaml && grep -n "returns the original creation result" docs/api/openapi.yaml` → both present.

- [ ] **Step 2: Register returns 201**

In the `/auth/register` operation, replace:
```
      responses:
        '200':
          description: Registration result.
```
with:
```
      responses:
        '201':
          description: Account created.
```

- [ ] **Step 3: Idempotency replay note**

Replace the `Idempotency-Key` parameter description:
```
          description: >-
            Optional idempotency key; a replay from the same principal within
            24 hours returns the original creation result.
```
with:
```
          description: >-
            Optional idempotency key; a replay from the same principal within
            24 hours returns 200 with the record's current representation
            (404 if since purged). The same key with a different body returns
            422. The first create returns 201.
```

- [ ] **Step 4: Post-verify**

Run: `grep -n "'201'\|current representation" docs/api/openapi.yaml` → both present; `grep -n "info" -A2 docs/api/openapi.yaml | grep "version: 1.0.0"` → unchanged.

- [ ] **Step 5: Commit**

```bash
git add docs/api/openapi.yaml
git commit -m "docs: openapi — register 201, idempotent-replay semantics"
```

---

### Task 13: CLAUDE.md + tracked skill files

**Files:**
- Modify: `CLAUDE.md`
- Modify (only if contradicted): `.claude/skills/auth-security-conventions/SKILL.md`, `.claude/skills/run-and-verify/SKILL.md`

- [ ] **Step 1: CLAUDE.md hard rule 6**

Replace:
```
6. Never log tokens, cookie values, presigned URLs, or JWT bodies — sole exceptions: the single-use setup/recovery tokens (BR-AUTH-11/12), logged once at warn with a 30-minute TTL.
```
with:
```
6. Never log tokens, cookie values, presigned URLs, or JWT bodies — sole exceptions: the single-use setup/recovery tokens (BR-AUTH-11/12), logged once at warn with a 30-minute TTL, emitted regardless of log level.
```

- [ ] **Step 2: Audit the two tracked skill files against this pass**

Read both files. Check for statements this pass falsifies: a sixteen-variable env count; an env-var list missing `CMS_END_USER_REGISTRATION`; the full public operator set including `contains`; registration described as always-on; media deletion order row-then-object-with-sweep-backstop; instance lock "fails immediately"; token logging without the log-level clause. Fix any contradiction minimally — do not otherwise expand the skills. If a file has no contradiction, leave it untouched and note that in the report.

- [ ] **Step 3: Post-verify**

Run: `grep -n "emitted regardless of log level" CLAUDE.md` → present. `grep -rn "sixteen" CLAUDE.md .claude/skills/auth-security-conventions/ .claude/skills/run-and-verify/` → no matches.

- [ ] **Step 4: Commit**

```bash
git add CLAUDE.md .claude/skills/auth-security-conventions/SKILL.md .claude/skills/run-and-verify/SKILL.md
git commit -m "docs: CLAUDE.md + skills — round-2 alignment (log-level clause, staleness check)"
```
(Only add the skill files if Step 2 changed them.)

---

### Task 14: Resolution Status appendix + acceptance sweep

**Files:**
- Modify: `docs/reviews/architecture-review-round2-2026-07-11.md`

- [ ] **Step 1: Append the Resolution Status table (verbatim) at the end of the review file**

```markdown
## Resolution Status (2026-07-11)

Every finding is dispositioned against `docs/superpowers/specs/2026-07-11-round2-review-remediation-design.md` (owner-approved, D2-1…D2-8); all named edits are committed in `docs/`.

| AR2 # | Sev | Finding | Disposition |
|---|---|---|---|
| AR2-1 | Blocker | Public write path had no content lifecycle; authors couldn't read their own creates; nothing public could publish. | Resolved by D2-1: `createStatus` grant key (12 §1/§3/§4) + owner-draft visibility (BR-API-2 amended; 02 invariant 3; 07 Public-API-create bullet); 12 §2 comments example updated. |
| AR2-2 | Blocker | No CORS design for `/api/v1`. | Resolved by D2-2: new BR-API-6; 04 §CORS; 05 §6 admin-surface line; no `Vary: Origin` needed (constant ACAO). |
| AR2-3 | High | Session-scoped instance lock: silent split-brain on connection loss; ghost-lock crash-loop on restart. | Resolved by D2-3: BR-RUNTIME-8 amended (dedicated connection, exit-on-loss watchdog, 120 s startup retry); 09 step 3 + keepalive runbook. |
| AR2-4 | High | Row-driven orphan sweep could never find the object orphaned by the row-first delete order — backstop claim false. | Resolved by D2-4: new BR-MEDIA-5 + `cms_media_deletions` outbox; 07 §Media Deletion rewritten; BR-MEDIA-2 wording reconciled; retention duty 8. |
| AR2-5 | High | Argon2id 64 MiB × unbounded concurrency = OOM vector. | Resolved by D2-5: BR-AUTH-3 amended — `min(4, NumCPU)` semaphore, 2 s wait, 429. |
| AR2-6 | High | `contains` (infix ILIKE) defeated the indexed-only public-filter rule. | Resolved by D2-6: BR-API-4 amended (public operator set excludes `contains`); pg_trgm V2 path in 03/11. |
| AR2-7 | Medium | Open registration with no kill-switch; no end-user admin surface. | Resolved: new BR-AUTH-14, `CMS_END_USER_REGISTRATION` (default disabled), `disabled_at`, `/api/admin/end-users`, F-34, UAC-1.7. |
| AR2-8 | Medium | Round-1 AR-15 (bounded limiter maps) claimed resolved but never landed. | Resolved: BR-RUNTIME-4 amended — bounded LRU, fixed entry cap. |
| AR2-9 | Medium | Idempotency semantics missing at the failure edges. | Resolved: 04 §Idempotent Creates completeness paragraph (same-txn insert, blocking concurrency, `request_hash`/422, 200-replay-current, 404-if-purged, 201-first); 07 `request_hash` column. |
| AR2-10 | Medium | No DDL `lock_timeout`; pool coupling unpriced. | Resolved: 03 lock_timeout paragraph; 09 constants rows (`lock_timeout` 5 s, pool max 10). |
| AR2-11 | Medium | Index naming/63-byte truncation unspecified; rename left stale names. | Resolved: 03 + 07 — `ix_<table>_<field>`, join-table truncation rule, renames rename dependent indexes. |
| AR2-12 | Medium | 03 (NUMERIC(p,s) matrix) vs 07 (bare NUMERIC) disagreement. | Resolved: 07 mapping (declared → `NUMERIC(p,s)`, else bare); 03 matrix rows (declared→bare Yes, bare→declared No). |
| AR2-13 | Medium | No anonymous read limit; `count=exact` open to anonymous. | Resolved: new BR-API-7 (300/min/IP; `count=exact` authenticated-only); 04/05 references. |
| AR2-14 | Medium | DB↔bucket point-in-time inconsistency after PITR unaddressed. | Resolved: 09 cross-store bullet + restore-drill spot-check. |
| AR2-15 | Low | Setup/recovery tokens invisible at `CMS_LOG_LEVEL=error`. | Resolved: 05/08/CLAUDE.md — emitted regardless of `CMS_LOG_LEVEL`. |
| AR2-16 | Low | BR-AUTH-2 column list omitted `csrf_hash`. | Resolved: BR-AUTH-2 amended. |
| AR2-17 | Low | 01 sketch omitted `Recover`. | Resolved: sketch updated to match 04's normative order. |
| AR2-18 | Low | BR-RUNTIME-3 omitted the instance-lock step. | Resolved: BR-RUNTIME-3 amended. |
| AR2-19 | Low | V2 publisher tick untagged in 09's V1 startup list. | Resolved: step 7 tagged (V2); 01 mirror-tagged. |
| AR2-20 | Low | Cookie `Max-Age=7d` vs 30-day absolute — sliding unstated. | Resolved: BR-AUTH-5 amended (24 h re-issue, 30-day cap); 05 §1. |
| AR2-21 | Low | No master-secret rotation runbook. | Resolved: 09 runbook — rotation = loss procedure, no dual-secret machinery. |
| AR2-22 | Low | CSP lacked `frame-ancestors`; no nosniff/Referrer-Policy. | Resolved: 06 Security Posture amended. |
| AR2-23 | Low | Advisory key values and sole-tenant keyspace assumption unstated. | Resolved: 09 — fixed constants `0x636D7300/01/02`, dedicated-database assumption. |
| AR2-24 | Low | No cap on collection count. | Resolved: 03 — 500 collections, 422 beyond. |
| AR2-25 | Low | `in` encoding and `contains`-on-non-text unspecified. | Resolved: 04 filtering grammar — comma-separated `in`; non-text `contains` → 422. |
| AR2-26 | Low | OpenAPI register returned 200; replay status unspecified. | Resolved: register → 201; replay semantics in the Idempotency-Key description. |
| AR2-27 | Low | ETag/304 semantics absent; edge `Vary`/`no-store` honoring unverifiable. | Resolved: BR-API-5 amended (If-None-Match → 304, strong body hash); 09 cache-contract smoke check runbook. |

All blocker- and High-class items are resolved in the committed documentation set as of 2026-07-11.
```

- [ ] **Step 2: Run the acceptance sweep (spec §Acceptance Criteria, all 10)**

```bash
grep -rn "createStatus" docs/architecture/12-access-rules.md docs/BUSINESS_RULES.md docs/architecture/07-data-model.md | head -5
grep -rn "sixteen" docs/
grep -n "BR-API-6\|BR-API-7\|BR-AUTH-14\|BR-MEDIA-5" docs/BUSINESS_RULES.md
grep -n "contains" docs/BUSINESS_RULES.md docs/architecture/04-api-layer.md
grep -rn "cms_media_deletions" docs/architecture/07-data-model.md docs/architecture/08-observability.md docs/BUSINESS_RULES.md
grep -rn "CMS_END_USER_REGISTRATION" docs/
grep -n "LRU" docs/BUSINESS_RULES.md
tail -5 docs/reviews/architecture-review-round2-2026-07-11.md
grep -rn "Last Updated:\*\* 2026-07-11" docs/architecture/*.md docs/BUSINESS_RULES.md docs/REQUIREMENTS.md | grep -v "1.2\|12-access"
grep -rn "Vary: Origin\|MASTER_SECRET_PREVIOUS" docs/ | grep -v "need no"
```
Expected: criterion 2 (sixteen) → no matches outside `docs/superpowers/` and `docs/reviews/` quotations; criterion 9 → only unedited docs (00, 10) may remain at 1.1; criterion 10 → no matches outside quotation contexts. Record each criterion's PASS/FAIL; fix any FAIL that is a missed mechanical edit, report anything structural.

- [ ] **Step 3: Commit**

```bash
git add docs/reviews/architecture-review-round2-2026-07-11.md
git commit -m "docs: round-2 review — Resolution Status table, all 27 findings dispositioned"
```

---

## Self-Review (done at write time)

- **Spec coverage:** D2-1 → Tasks 1/2/3/4/5/7; D2-2 → Tasks 1/5/8; D2-3 → Tasks 1/9/10; D2-4 → Tasks 1/2(none)/4/7/8(none)/10; D2-5 → Tasks 1/8; D2-6 → Tasks 1/4/5/6/11; §7 sweep → Tasks 1/3/5/6/7/8/9; §8 sweep → Tasks 1/5/6/9/10/11/12/13; disposition table → Task 14. Every edit-matrix row has a task; 00-README.md and 10-project-structure.md are intentionally untouched (spec: verify-only, covered by Task 14's sweep).
- **Placeholder scan:** none.
- **Consistency:** identifiers (BR-API-6/7, BR-AUTH-14, BR-MEDIA-5, `createStatus`, `cms_media_deletions`, `disabled_at`, `request_hash`, `ix_<table>_<field>`, 300/min, 120 s, 5 s, pool 10, 500 collections) are spelled identically across tasks.
