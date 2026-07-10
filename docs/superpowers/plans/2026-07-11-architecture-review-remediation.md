# Architecture-Review Remediation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute the documentation-only remediation of `docs/superpowers/specs/2026-07-11-architecture-review-remediation-design.md` — six new business rules, eight amended rules, one new architecture document, an OpenAPI skeleton, and coordinated edits across all fifteen existing docs — so V1 implementation starts with zero open design questions.

**Architecture:** Identifier-bearing documents change first (BUSINESS_RULES → new doc 12 → REQUIREMENTS), then each architecture doc lands its edits, then cross-cutting files (CLAUDE.md, review appendix, OpenAPI skeleton), then a final acceptance sweep runs every grep in the spec's acceptance criteria. Each task is one document (or one tightly coupled pair) and is independently reviewable.

**Tech Stack:** Markdown edits only. Verification via `grep`/`rg`. No Go, Svelte, SQL, or Makefile changes — those belong to V1 implementation, not this pass.

## Global Constraints

- **The spec is the authoritative content source:** `docs/superpowers/specs/2026-07-11-architecture-review-remediation-design.md`. Read the cited spec section before executing a task; where this plan says "per spec §N", that section contains the normative content to transcribe.
- **Docs only.** Touch nothing outside `docs/` except the single CLAUDE.md edit in Task 15.
- **Authority chain holds:** `BUSINESS_RULES.md` > `docs/architecture/*`. Never introduce wording in an architecture doc that contradicts a rule written in Task 1.
- **Identifier wording is load-bearing** (tests will trace to it). Copy rule text from this plan verbatim; do not paraphrase.
- **Env vars: exactly 16** after Task 1: the current 14 with `CMS_SESSION_SECRET` renamed `CMS_MASTER_SECRET`, plus `CMS_RECOVERY_EMAIL` and `CMS_TRUSTED_PROXY_CIDRS`. `CMS_SESSION_SECRET` must not survive anywhere in `docs/`.
- **Version headers:** every modified doc sets `**Version:** 1.1 · **Last Updated:** 2026-07-11`; the new doc 12 starts at `1.0 · 2026-07-11`.
- **EC citation convention (01-system-overview):** any new section resolving an edge case cites it inline as *(Resolves EC-n.)*.
- **Git: commits are authorized by the user (2026-07-11) with plain messages and NO Co-Authored-By trailer.** "Checkpoint" in a task means: `git add` exactly the files that task touched, commit with a plain `docs(<area>): <summary>` message, and report the change to the user. Never stage unrelated untracked files (`.claude/skills/system-design/`, `prompt.md`).
- Numbers that recur across tasks (use exactly these): reset/setup/recovery token TTL **30 minutes**; cache header `public, s-maxage=60, stale-while-revalidate=60`; offset cap **10,000 → 422**; RPO **≤ 5 min**, RTO **≤ 1 h**; per-field cap **1 MiB**; record-write body cap **5 MiB**, other routes **64 KiB**; `statement_timeout` **10 s** (collection queries) / **60 s** (schema transactions); server timeouts ReadHeader 5 s / Read 30 s / Write 30 s / Idle 120 s; idempotency window **24 h**; Argon2id **64 MiB, 3 iterations, parallelism 2**; password minimums **10 admin / 8 end-user**.

---

### Task 1: BUSINESS_RULES.md — rule delta, env table, EC note

**Files:**
- Modify: `docs/BUSINESS_RULES.md`

**Interfaces:**
- Produces: identifiers every later task cites — BR-RBAC-7, BR-AUTH-12, BR-AUTH-13, BR-RUNTIME-8, BR-API-4, BR-API-5; amended BR-AUTH-10/11, BR-LIFE-7/9, BR-API-1, BR-RUNTIME-6, BR-SCHEMA-7, BR-RBAC-3/4; env names `CMS_MASTER_SECRET`, `CMS_RECOVERY_EMAIL`, `CMS_TRUSTED_PROXY_CIDRS`.

- [ ] **Step 1: Pre-verify the defects exist**

Run: `grep -c "CMS_SESSION_SECRET" docs/BUSINESS_RULES.md` → Expected: `2` (env table + none elsewhere; any non-zero confirms the rename is pending)
Run: `grep -n "greater than 10,000 with \`400\`" docs/BUSINESS_RULES.md` → Expected: one hit in BR-API-1
Run: `grep -n "drain drops zero accepted requests" docs/BUSINESS_RULES.md` → Expected: one hit in BR-RUNTIME-6

- [ ] **Step 2: Amend existing rules (verbatim replacements)**

Replace the corresponding sentences/rules with exactly:

- **BR-RUNTIME-6.** `Shutdown drains in-flight requests within a 15-second window; requests completing within the window are never dropped; requests exceeding it are force-closed, logged, and counted.` (Enforcement line unchanged.)
- **BR-LIFE-7.** `Record updates require the current `version` value; a mismatch returns `409 Conflict` and writes nothing. Every save advances the live row's `version` and `updated_at` via compare-and-set — including saves that create pending drafts; for published records the content columns remain frozen until publish.` *Enforcement:* unchanged (`lifecycle.Service.Save` — compare-and-set `WHERE version = $expected`).
- **BR-LIFE-9 (V2).** Replace "on startup the system publishes every record whose `publish_at` elapsed during downtime" with: `the publisher's first tick, immediately after the listener opens, publishes every record whose `publish_at` elapsed during downtime.`
- **BR-AUTH-10.** Append: `The stored `private_pem` is AES-256-GCM ciphertext under an HKDF key derived from `CMS_MASTER_SECRET`; `public_pem` remains plaintext. Issued JWTs carry a `kid` header naming the signing key row.`
- **BR-AUTH-11.** Replace "The token dies on use or process exit." with: `The token dies on use, after 30 minutes, or on process exit.`
- **BR-API-1.** Replace `with \`400\`` with `with \`422 validation_failed\``.
- **BR-SCHEMA-7.** Append: `Dropping a collection deletes its table, its `cms_fields` rows, and its entire `cms_revisions` history in one transaction; the typed confirmation states this explicitly. (Field drops retain revision data; collection drops do not — the asymmetry is deliberate.)`
- **BR-RBAC-3.** Replace with: `A missing access rule denies for the governed classes (`editor`, `contributor`, `viewer`, `end_user`, `anonymous`). `super_admin` and `admin` hold an implicit full grant on content actions; no other default-allow path exists.` (Enforcement line unchanged.)
- **BR-RBAC-4.** Replace `hideFromRoles` / `readOnlyForRoles` with `hideFrom` / `readOnlyFor`, and append: `Audience lists draw from the closed set: the five roles plus `end_user`, `anonymous`, `api_key`.`

- [ ] **Step 3: Add new rules (verbatim)**

Under §5 Access Control:
- **BR-RBAC-7.** `Access-rule objects conform to the closed grant-matrix schema of `docs/architecture/12-access-rules.md`; writes violating it fail with `422`, and rules failing validation at evaluation time deny (fail closed, N-11).` *Enforcement:* `httpapi/admin` rule validator + `access.Evaluator` fail-closed branch.

Under §4 Authentication:
- **BR-AUTH-12.** `When `CMS_RECOVERY_EMAIL` names an existing `cms_users` row at startup, the system generates a 256-bit single-use recovery token, logs it once at `warn`, and enables `/recover`; consuming it resets that user's password and revokes their sessions. The token dies on use, after 30 minutes, or on process exit; the route returns 404 otherwise.` *Enforcement:* `auth.Recovery` at startup + the `/recover` handler.
- **BR-AUTH-13.** `Password-reset tokens store hashed, expire in 30 minutes, and are single-use; confirmation revokes every refresh-token family of the affected user. The binary never sends email; delivery belongs to the consuming application.` *Enforcement:* `auth.ResetService` + `cms_reset_tokens`.

Under §1 Tenancy & Runtime:
- **BR-RUNTIME-8.** `Exactly one process serves at a time: before opening the listener, the binary acquires a process-lifetime advisory lock (session-scoped, distinct from the migration and schema keys); a second process fails startup with a clear log line. *(Resolves EC-16.)*` *Enforcement:* `app.Run` — `pg_advisory_lock` on the instance key, held until exit.

Under §7 API Conduct:
- **BR-API-4.** `Public-scope list queries accept `filter`/`sort` only on fields marked `indexed` or `unique`; violations return `422` naming the field. Admin and trash scopes accept any schema field.` *Enforcement:* `query.Builder` scope-aware field validation.
- **BR-API-5.** `Anonymous public reads carry `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` and a strong `ETag`; any request bearing `Authorization` or a cookie receives `Cache-Control: no-store` without exception.` *Enforcement:* `httpapi` response headers on public routes.

- [ ] **Step 4: Env table → 16 rows**

Rename the `CMS_SESSION_SECRET` row to:
`| \`CMS_MASTER_SECRET\` | — | Root secret for at-rest encryption of system key material (BR-AUTH-10). |`
Add:
`| \`CMS_RECOVERY_EMAIL\` | unset | When set at startup, enables single-use super-admin recovery (BR-AUTH-12). |`
`| \`CMS_TRUSTED_PROXY_CIDRS\` | empty | Peers within these CIDRs are trusted to append \`X-Forwarded-For\` (05 §5). |`

- [ ] **Step 5: Edge-case coverage note**

In the "Edge-Case Coverage (Batch 1)" section, append to the closing parenthetical: `BR-RUNTIME-8 resolves EC-16.`

- [ ] **Step 6: Bump header, verify**

Set `**Version:** 1.1 · **Last Updated:** 2026-07-11`.
Run: `grep -c "CMS_SESSION_SECRET" docs/BUSINESS_RULES.md` → Expected: `0`
Run: `grep -c "^| \`" docs/BUSINESS_RULES.md` → Expected: `16` (env rows)
Run: `grep -c "BR-RBAC-7\|BR-AUTH-12\|BR-AUTH-13\|BR-RUNTIME-8\|BR-API-4\|BR-API-5" docs/BUSINESS_RULES.md` → Expected: ≥ 6

- [ ] **Step 7: Checkpoint** — report the rule delta to the user; do not commit.

---

### Task 2: Create docs/architecture/12-access-rules.md

**Files:**
- Create: `docs/architecture/12-access-rules.md`

**Interfaces:**
- Consumes: BR-RBAC-3/4/7 wording from Task 1.
- Produces: the grant-object keys later tasks cite — `minRole`, `minRoleOwn`, `endUsers` (`"own"|"all"`), `anonymous`; role lattice `viewer < contributor < editor < admin < super_admin`; scope shape `{"collections": [{"id", "actions", "draftAccess"}], "passwordReset": false}`; audience set (five roles + `end_user`, `anonymous`, `api_key`).

- [ ] **Step 1: Write the document** with header `# 12 — Access Rules` / `**Version:** 1.0 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal`, transcribing spec §1.1–§1.6 in full and in order: (1) grant-matrix schema table with the four keys and their exact semantics; (2) the three worked examples verbatim (public blog, comments, ingestion target) as normative; (3) the six-step evaluation algorithm including the implicit `super_admin`/`admin` grant, the `publish` floor at `editor` (BR-LIFE-3), api_key scope resolution with `draftAccess` and revoked-key → `anonymous`; (4) write-time validation rules (unknown keys/roles, `minRoleOwn > minRole`, `anonymous` outside `read` → 422) and the fail-closed read branch; (5) field-rule audiences `hideFrom`/`readOnlyFor`; (6) API-key scope shape including `passwordReset`. Close with a "Rules resolved here" list: BR-RBAC-2, BR-RBAC-3, BR-RBAC-4, BR-RBAC-6, BR-RBAC-7.

- [ ] **Step 2: Verify**

Run: `grep -c "minRoleOwn" docs/architecture/12-access-rules.md` → Expected: ≥ 3
Run: `grep -c "^\`\`\`" docs/architecture/12-access-rules.md` → Expected: ≥ 6 (three fenced examples)
Run: `grep -c "passwordReset" docs/architecture/12-access-rules.md` → Expected: ≥ 1

- [ ] **Step 3: Checkpoint** — report; do not commit.

---

### Task 3: REQUIREMENTS.md — new F/N/UAC items and amendments

**Files:**
- Modify: `docs/REQUIREMENTS.md`

**Interfaces:**
- Consumes: BR-AUTH-12/13, BR-API-4/5 identifiers (Task 1).
- Produces: F-32, F-33, N-12, N-13, UAC-1.6 — cited by Tasks 12–13.

- [ ] **Step 1: Pre-verify** — Run: `grep -n "the prompt" docs/REQUIREMENTS.md` → Expected: one hit (F-2).

- [ ] **Step 2: Apply amendments (verbatim)**

- **F-2:** replace "defined in the prompt's field-type reference" with `defined in `docs/architecture/07-data-model.md``.
- **F-15:** append `Admin lists additionally support keyset cursor pagination from V1.`
- **F-16:** append `V1 audit durability equals shipped stdout logs; queryable persistence arrives in V2 (accepted limitation).`
- **F-27:** replace with `**F-27 (V2).** The public API exposes the V1 keyset-cursor mechanism alongside capped offset pagination.`

- [ ] **Step 3: Add new items (verbatim)**

V1 list: `**F-32 (V1).** Account recovery and password reset: env-gated super-admin recovery mode (BR-AUTH-12); admin-issued one-time reset tokens; end-user reset via API-key-gated request plus public confirm, revoking all refresh-token families on success (BR-AUTH-13). The binary never sends email.`
V2 list: `**F-33 (V2).** GDPR-class erasure: end-user hard delete with token-family revocation, `created_by` anonymization to a tombstone UUID, and a revision-redaction capability in the retention job, with documented limitations.`
§4: `**N-12.** RPO ≤ 5 minutes and RTO ≤ 1 hour via WAL-based point-in-time recovery; scheduled `pg_dump` remains the portable second copy; a restore drill precedes V1 release.`
`**N-13.** The service is single-instance: upgrades cost drain-plus-startup downtime and the availability class is ~99.5%; high availability is out of scope.`
§6 V1: `**UAC-1.6.** An API consumer whose key carries `passwordReset` requests a reset for an end user and receives a token; confirmation sets the new password and revokes every refresh-token family; separately, a recovery-mode start (`CMS_RECOVERY_EMAIL`) logs a single-use token that resets a locked-out super admin and dies on use.`

- [ ] **Step 4: Bump header, verify**

Run: `grep -c "the prompt" docs/REQUIREMENTS.md` → Expected: `0`
Run: `grep -c "F-32\|F-33\|N-12\|N-13\|UAC-1.6" docs/REQUIREMENTS.md` → Expected: ≥ 5

- [ ] **Step 5: Checkpoint** — report; do not commit.

---

### Task 4: 01-system-overview.md + 00-README.md

**Files:**
- Modify: `docs/architecture/01-system-overview.md`, `docs/architecture/00-README.md`

- [ ] **Step 1: 01 edits** — In the Edge-Case Register table add: `| EC-16 | Accidental second running instance | \`09-deployment.md\` (BR-RUNTIME-8) |`. In Request Flows, re-root public examples under `/api/v1` and mention the `Idempotency-Key` option on public creates. In Lifecycle Summary: startup ends `→ instance lock → HTTP listener; the publisher's first tick runs the missed-schedule catch-up (BR-LIFE-9)`; add `/readyz` alongside `/healthz`. In the Consumer Classes table, update P-6/P-7 surfaces to `/api/v1/...`.
- [ ] **Step 2: 00 edits** — Document Map gains `| \`12-access-rules.md\` | The access-control grant matrix: schema, evaluation algorithm, audiences, API-key scopes. |` and a `docs/api/openapi.yaml` pointer line. Reading order: insert `12` after `05` in the full pass and the backend pass.
- [ ] **Step 3: Verify** — `grep -c "EC-16" docs/architecture/01-system-overview.md` → ≥ 1; `grep -c "12-access-rules" docs/architecture/00-README.md` → ≥ 2. Bump both headers.
- [ ] **Step 4: Checkpoint.**

---

### Task 5: 02-core-interfaces.md

**Files:**
- Modify: `docs/architecture/02-core-interfaces.md`

- [ ] **Step 1: Edits** — `query.Builder`: `Paginate(p Page)` gains cursor support (`Page` carries either offset or an opaque cursor of sort key + `id`; both → error); invariants list gains `7. ScopePublic accepts filter/sort only on indexed or unique fields (BR-API-4); cursor pagination preserves the id tiebreaker and limit clamps.` `content.Document`: append `Any single field value caps at 1 MiB; larger values reject with a field-level error.` `access.Evaluator`: append `Grant semantics, audiences, and scope shapes are normative in 12-access-rules.md (BR-RBAC-7).` and update `FieldRules` prose to `hideFrom`/`readOnlyFor`. Auth services table: `auth.JWTService` guarantees gain `kid header (BR-AUTH-10)`; add row `| \`auth.ResetService\` | \`Request(kind, userID) (plaintextOnce, ResetToken)\` · \`Confirm(token, newPassword) error\` | Hashed at rest, 30-min TTL, single-use; Confirm revokes all refresh-token families (BR-AUTH-13). |`
- [ ] **Step 2: Verify** — `grep -c "ResetService" docs/architecture/02-core-interfaces.md` → ≥ 2; `grep -c "hideFromRoles" docs/architecture/02-core-interfaces.md` → `0`. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 6: 03-dynamic-schema.md

**Files:**
- Modify: `docs/architecture/03-dynamic-schema.md`

- [ ] **Step 1: Pre-verify** — `grep -n "reads proceed via MVCC" docs/architecture/03-dynamic-schema.md` → one hit (EC-12 section).
- [ ] **Step 2: Edits** — EC-1 item 2: append `New reads also queue: DDL that rewrites the table holds ACCESS EXCLUSIVE, which conflicts with ACCESS SHARE.` EC-12 section: delete "reads proceed via MVCC", replace with `The rebuild holds ACCESS EXCLUSIVE: reads and writes to that collection stall for its duration (~seconds at the 100k-row design point; the audit event records the duration — schedule heavy changes off-peak).` Append the documented V2 alternative: trigger-maintained tsvector + `CREATE INDEX CONCURRENTLY` outside the schema transaction with a reconciliation step and bounded staleness window. Destructive Changes section: add the DropCollection cascade sentence matching amended BR-SCHEMA-7 (Task 1 wording).
- [ ] **Step 3: Verify** — `grep -c "proceed via MVCC" docs/architecture/03-dynamic-schema.md` → `0`; `grep -c "ACCESS EXCLUSIVE" docs/architecture/03-dynamic-schema.md` → ≥ 2. Bump header.
- [ ] **Step 4: Checkpoint.**

---

### Task 7: 04-api-layer.md

**Files:**
- Modify: `docs/architecture/04-api-layer.md`

- [ ] **Step 1: Edits** — Route map: public rows become `/api/v1/collections/{slug}/records` and `/api/v1/auth/*` (register, login, refresh, logout, `password-reset/request`, `password-reset/confirm`); add `/recover` (gated by BR-AUTH-12) and `/readyz`; admin rows unchanged; add a line: `The envelope and error registry are v1-stable and evolve additively only.` Pagination: offset > 10,000 → `422 validation_failed`; add V1 cursor paragraph (admin lists; opaque base64 sort-key+id; `cursor` and `offset` mutually exclusive → 422; public exposure is F-27/V2). Envelope: `meta.pagination.total` appears only with `?count=exact`. Filtering: public scope indexed/unique-only per BR-API-4 with 422 naming the field. New section **Caching (BR-API-5)** with the exact header values and the credentialed-⇒-`no-store` invariant plus `Vary: Authorization`; note V2 purge-on-publish lengthens TTLs and V1 accepts ≤60 s propagation. New section **Idempotent Creates**: optional `Idempotency-Key` (≤128 bytes) on public/API-key creates; same key + principal within 24 h returns the original result (`cms_idempotency_keys`, 07). Relation Expansion: add `Expansion resolves each relation field with a single batched IN query per field, never per-row lookups.` Upload flow: presign validates MIME against the closed compile-time V1 allowlist (raster images, video, audio, PDF); `Content-Type` is signed into the presign policy and re-verified at finalize HEAD. Body caps: 5 MiB record writes, 64 KiB elsewhere.
- [ ] **Step 2: Verify** — `grep -c "/api/v1" docs/architecture/04-api-layer.md` → ≥ 4; `grep -c "Idempotency-Key" docs/architecture/04-api-layer.md` → ≥ 2; `grep -c "s-maxage=60" docs/architecture/04-api-layer.md` → ≥ 1. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 8: 05-auth-security.md

**Files:**
- Modify: `docs/architecture/05-auth-security.md`

- [ ] **Step 1: Edits** — §1: replace the `CMS_SESSION_SECRET` sentence with `Session tokens are random 256-bit values verified by hashed lookup; no signing is involved. \`CMS_MASTER_SECRET\` is the root secret for at-rest encryption of system key material (§3) and plays no role in sessions.`; add session-ID + CSRF rotation on login; password minimums 10/8, length-only; pin Argon2id parameters (64 MiB, 3 iterations, parallelism 2). New §1 subsection **Recovery Mode (BR-AUTH-12)** with the Task 1 rule semantics and admin-issued one-time reset tokens. §3: `kid` header; `private_pem` AES-256-GCM under HKDF(`CMS_MASTER_SECRET`); master-secret-loss blast radius note (regenerate keypair; ≤15-min JWTs re-issue via refresh); new subsection **Password Reset (BR-AUTH-13)** describing request (key with `passwordReset: true`; unknown email → 404 — trusted caller, enumeration posture applies to public endpoints only) and confirm (revokes all families). §4: point grant semantics at doc 12; rename field-rule keys. §5: replace the heuristic with the CIDR rule: peer in loopback/RFC1918/`CMS_TRUSTED_PROXY_CIDRS` → rightmost non-trusted XFF entry; otherwise socket address, XFF ignored; empty variable = prior behavior; failure mode = per-proxy-IP limiting (fails closed). Rate limiting: `register` and `refresh` adopt BR-AUTH-6's numbers; uniform errors + constant Argon2id work on unknown email; document the per-email targeted-lockout trade-off. §6 additions: SSRF (V2 webhooks — deny private/link-local/loopback, re-resolve DNS, no redirects-to-private), user enumeration, targeted rate-limit lockout, malicious media hosting (mitigated by MIME allowlist + origin isolation), DB-read key theft (mitigated by at-rest encryption).
- [ ] **Step 2: Verify** — `grep -c "CMS_SESSION_SECRET" docs/architecture/05-auth-security.md` → `0`; `grep -c "CMS_TRUSTED_PROXY_CIDRS" docs/architecture/05-auth-security.md` → ≥ 1; `grep -c "BR-AUTH-12\|BR-AUTH-13" docs/architecture/05-auth-security.md` → ≥ 2. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 9: 06-admin-ui.md

**Files:**
- Modify: `docs/architecture/06-admin-ui.md`

- [ ] **Step 1: Edits** — Security Posture: CSP becomes `default-src 'self'; img-src 'self' <media domain>; media-src 'self' <media domain>` with a sentence naming `R2_PUBLIC_BUCKET_URL` as the media domain and the origin-isolation requirement (09). Screens table: Users & API keys row gains one-time reset-token issuance (BR-AUTH-13) and the `passwordReset` key capability toggle; add `/recover` row (`Shown only when recovery mode is active (BR-AUTH-12); accepts the logged single-use token; 404 otherwise.`); Access rules row now says the editor renders the grant matrix of doc 12 (minRole/minRoleOwn/endUsers/anonymous per action) with the per-role preview. Trash row: restoring a published record warns it becomes publicly visible immediately. Content lists: cursor-aware next/prev beyond the 10k offset ceiling; tables request `?count=exact` where totals are shown.
- [ ] **Step 2: Verify** — `grep -c "img-src" docs/architecture/06-admin-ui.md` → ≥ 1; `grep -c "/recover" docs/architecture/06-admin-ui.md` → ≥ 1. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 10: 07-data-model.md

**Files:**
- Modify: `docs/architecture/07-data-model.md`

- [ ] **Step 1: Pre-verify** — `grep -n "the live row does not change" docs/architecture/07-data-model.md` → one hit.
- [ ] **Step 2: Edits** — Live-Table/Revisions Contract: replace the "Published record" bullet's contradiction with `Edits advance the live row's system columns (\`version\`, \`updated_at\`) via compare-and-set and insert a revision; the content columns remain frozen until publish (BR-LIFE-7).` System tables: add `cms_reset_tokens` (`id`, `user_kind` admin|end_user, `user_id`, `token_hash` unique, `expires_at`, `used_at`, `created_by`, `created_at`) and `cms_idempotency_keys` (`key_hash`, `principal_id`, `record_id`, `created_at`; unique `(key_hash, principal_id)`); reserve `cms_webhook_deliveries` (V2 outbox) in the V2 sentence. Indexes: `indexed` fields get composite `(field, id)` B-trees serving the mandatory tiebreaker. Retention: duties 4–6 (expired sessions per BR-AUTH-5; used/expired reset tokens; rotated/revoked refresh tokens > 30 days) and idempotency rows > 24 h. New subsections: **Media Deletion** (destructive-gated; 409 while referenced via FK RESTRICT; row deleted then object; orphan sweep as crash-window backstop); **Erasure (V2, F-33)** semantics per spec §7.3; **Admin deletion** (sessions and reset tokens cascade; `created_by` persists by design; audit stores actor snapshots). Add: `publish_at` comparisons use database `now()`; single field values cap at 1 MiB in `Document.Set`.
- [ ] **Step 3: Verify** — `grep -c "does not change" docs/architecture/07-data-model.md` → `0`; `grep -c "cms_reset_tokens\|cms_idempotency_keys" docs/architecture/07-data-model.md` → ≥ 2. Bump header.
- [ ] **Step 4: Checkpoint.**

---

### Task 11: 08-observability.md

**Files:**
- Modify: `docs/architecture/08-observability.md`

- [ ] **Step 1: Edits** — Health section: `/healthz` = liveness (200 once started; 503 during drain); new `/readyz` = readiness (DB ping; proxies route on it, supervisors restart on `/healthz`). Replace "It is the only unauthenticated non-SPA endpoint" with `\`/healthz\`, \`/readyz\`, \`/api/v1/auth/*\`, and anonymous public reads are the unauthenticated surfaces.` Secrets rule: append `Sole exceptions: the single-use setup (BR-AUTH-11) and recovery (BR-AUTH-12) tokens, logged once at \`warn\` with a 30-minute TTL.` Job telemetry: tickers wrap each tick in panic recovery (a panicking job logs at `error` and skips to the next tick); retention counts gain sessions/reset-tokens/refresh-tokens/idempotency purges.
- [ ] **Step 2: Verify** — `grep -c "readyz" docs/architecture/08-observability.md` → ≥ 2; `grep -c "only unauthenticated non-SPA" docs/architecture/08-observability.md` → `0`. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 12: 09-deployment.md

**Files:**
- Modify: `docs/architecture/09-deployment.md`

- [ ] **Step 1: Edits** — Startup list becomes: validate config → migrations under migration lock → **acquire the process-lifetime instance lock (BR-RUNTIME-8); fail fast if held** *(Resolves EC-16.)* → schema cache → bootstrap/recovery token steps (BR-AUTH-11/12) → listener (`/readyz` 200) → publisher first tick runs catch-up (BR-LIFE-9). Configuration: sixteen variables, table matches BUSINESS_RULES. New **Timeouts** table with the Global Constraints values (server timeouts, request deadline 25 s, statement timeouts, body caps). Proxy contract: requirement 2 references `CMS_TRUSTED_PROXY_CIDRS` with a Cloudflare-ranges example line and the fails-closed note; add requirement 4: the edge must honor origin cache headers on `/api/*` (BR-API-5). New **Media origin & CORS**: bucket CORS policy allowing `PUT` from the admin origin (no credentialed wildcard) — a UAC-1.5 prerequisite; `R2_PUBLIC_BUCKET_URL` must not share a registrable domain with the admin origin. Backup: PITR required (managed Postgres or pgBackRest/WAL-G), `pg_dump` as portable second copy, N-12 targets stated. New **Runbooks** section: JWT key rotation (successor `kid`, verify-both window ≥ 15 min, retire), super-admin recovery, restore drill (required before V1 release), hot-DDL guidance (03). Availability statement per N-13.
- [ ] **Step 2: Verify** — `grep -c "CMS_TRUSTED_PROXY_CIDRS" docs/architecture/09-deployment.md` → ≥ 1; `grep -c "Runbook" docs/architecture/09-deployment.md` → ≥ 1; `grep -c "PITR\|point-in-time" docs/architecture/09-deployment.md` → ≥ 1. Bump header.
- [ ] **Step 3: Checkpoint.**

---

### Task 13: 10-project-structure.md + 11-roadmap.md

**Files:**
- Modify: `docs/architecture/10-project-structure.md`, `docs/architecture/11-roadmap.md`

- [ ] **Step 1: 10 edits** — Package Rules table: add `depguard` (or equivalent) CI enforcement of the import rules. Makefile table: add `| bench | Seeded 100k-row database asserting N-3/N-4; required before release gates. |` Layout: add `docs/api/` (openapi.yaml).
- [ ] **Step 2: 11 edits** — V1 scope append: recovery & reset flows (F-32), keyset cursors on admin lists, the BR-API-5 cache contract, idempotent creates, `make bench` in the V1 gate row. V2 scope append: GDPR erasure (F-33), webhook outbox table (`cms_webhook_deliveries`) with SSRF egress rules, purge-on-publish lengthening cache TTLs. Remove/replace the phrase "match the generation contract verbatim" with "are authoritative". V1 gate row: `UAC-1.1 … UAC-1.6 pass end-to-end; \`make bench\` meets N-3/N-4; CI \`trace\` green.`
- [ ] **Step 3: Verify** — `grep -c "generation contract" docs/architecture/11-roadmap.md` → `0`; `grep -c "bench" docs/architecture/10-project-structure.md` → ≥ 1. Bump both headers.
- [ ] **Step 4: Checkpoint.**

---

### Task 14: docs/api/openapi.yaml skeleton

**Files:**
- Create: `docs/api/openapi.yaml`

- [ ] **Step 1: Write the skeleton** — `openapi: 3.1.0`; `info` (title `golang-cms public API`, version `1.0.0`); `servers: [{url: /api/v1}]`; paths stubs with one-line summaries whose responses `$ref` the components defined below: `/collections/{slug}/records` (GET list with `limit`, `offset`, `cursor`, `count`, `filter`, `sort`, `expand` params; POST with `Idempotency-Key` header param), `/collections/{slug}/records/{id}` (GET), `/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/password-reset/request`, `/auth/password-reset/confirm`; `components.schemas`: `Envelope` (`data`, `meta.pagination{limit,offset,total?,next_cursor?}`), `Error` (`error{code,message,details}` with the eight-code enum: `validation_failed`, `unauthorized`, `forbidden`, `not_found`, `conflict`, `rate_limited`, `payload_too_large`, `internal`); `components.securitySchemes`: `apiKey` (http bearer, `cms_` prefix noted), `jwt` (http bearer). Every stub carries `description: Normative behavior in docs/architecture/04-api-layer.md`.
- [ ] **Step 2: Verify** — `grep -c "password-reset" docs/api/openapi.yaml` → `2`; `grep -c "validation_failed" docs/api/openapi.yaml` → ≥ 1.
- [ ] **Step 3: Checkpoint.**

---

### Task 15: CLAUDE.md + review-doc resolution appendix

**Files:**
- Modify: `CLAUDE.md`, `docs/reviews/architecture-review-2026-07-11.md`

- [ ] **Step 1: CLAUDE.md** — Documentation Map table: add `| \`docs/architecture/12-access-rules.md\` | Grant-matrix schema, evaluation algorithm, audiences, API-key scopes |`. Hard rule 6 becomes: `Never log tokens, cookie values, presigned URLs, or JWT bodies — sole exceptions: the single-use setup/recovery tokens (BR-AUTH-11/12), logged once at warn with a 30-minute TTL.`
- [ ] **Step 2: Review appendix** — Append `## Resolution Status (2026-07-11)` to the review doc: one table row per AR-1…AR-47 with disposition (`Resolved by spec §N` / `Deferred to V2 (F-33, outbox)` / `Accepted trade-off, documented`), pointing at `docs/superpowers/specs/2026-07-11-architecture-review-remediation-design.md`. Every blocker AR-1…AR-5 must map to a spec section.
- [ ] **Step 3: Verify** — `grep -c "12-access-rules" CLAUDE.md` → ≥ 1; `grep -c "Resolution Status" docs/reviews/architecture-review-2026-07-11.md` → `1`.
- [ ] **Step 4: Checkpoint.**

---

### Task 16: Final acceptance sweep

**Files:** none (verification only)

- [ ] **Step 1: Run every spec acceptance criterion**

```bash
grep -rn "CMS_SESSION_SECRET" docs/ && echo FAIL || echo PASS          # criterion 1/2
grep -c "^| \`" docs/BUSINESS_RULES.md                                  # expect 16
test -f docs/architecture/12-access-rules.md && echo PASS               # criterion 3
grep -rn "proceed via MVCC" docs/ && echo FAIL || echo PASS             # criterion 4
grep -n "/api/collections" docs/architecture/04-api-layer.md            # expect 0 hits (all /api/v1) — criterion 5
grep -rn "the prompt" docs/BUSINESS_RULES.md docs/REQUIREMENTS.md docs/architecture/ && echo FAIL || echo PASS  # criterion 6
grep -c "Resolved by spec" docs/reviews/architecture-review-2026-07-11.md   # ≥ 5 — criterion 7
grep -n "does not change" docs/architecture/07-data-model.md && echo FAIL || echo PASS  # criterion 8
```

- [ ] **Step 2: Cross-reference sweep** — confirm doc 12 is referenced from 02, 05, 06, CLAUDE.md, and 00-README; confirm every new BR identifier (BR-RBAC-7, BR-AUTH-12/13, BR-RUNTIME-8, BR-API-4/5) appears in at least one architecture doc; confirm all touched docs carry `Last Updated: 2026-07-11`.
- [ ] **Step 3: Final checkpoint** — report the full sweep results to the user; the user reviews and commits.
