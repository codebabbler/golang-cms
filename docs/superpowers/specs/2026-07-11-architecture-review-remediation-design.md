# Architecture-Review Remediation — Design Spec

**Date:** 2026-07-11 · **Status:** awaiting user approval · **Input:** `docs/reviews/architecture-review-2026-07-11.md` (findings AR-1…AR-47) · **Decisions:** made interactively with the owner on 2026-07-11; all eight recommendations accepted.

## Goal

Resolve every blocker and decision-heavy High finding from the architecture review as **documentation changes only** — amended business rules, new rules, corrected contracts, and one new architecture document — so V1 implementation starts with zero open design questions. No code exists yet; this spec's "implementation" is a coordinated editing pass across `docs/`.

## Non-Goals

- Writing any Go/Svelte code or migrations.
- Re-opening decisions the review endorsed (single tenant, single process, two dependencies, REST-only, no plugin runtime).
- Addressing V3 findings beyond reserving constraints already noted in the review (Stripe raw-body/idempotency).

## Decision Log

| # | Finding | Decision (owner-approved) |
|---|---|---|
| D-1 | AR-1 | Access rules = **declarative grant matrix**, closed vocabulary, new doc `12-access-rules.md` |
| D-2 | AR-5 | Super-admin rescue = **env-gated recovery mode** (`CMS_RECOVERY_EMAIL`); end-user reset = **app-mediated reset API**; admin resets = super-admin-issued one-time tokens |
| D-3 | AR-4 | Client-IP trust = **`CMS_TRUSTED_PROXY_CIDRS`** (empty default preserves current heuristic) |
| D-4 | AR-16/17 | **`CMS_SESSION_SECRET` → `CMS_MASTER_SECRET`**; RS256 private key AES-256-GCM-encrypted at rest; `kid` claim; rotation runbook |
| D-5 | AR-10 | Filter/sort policy: **public scope indexed/unique fields only; admin scope any field**; counts opt-in; statement timeouts |
| D-6 | AR-7 | **PITR required; RPO ≤ 5 min, RTO ≤ 1 h**; `pg_dump` retained as portable second copy |
| D-7 | AR-30 | GDPR erasure: **semantics specified now, implemented in V2** |
| D-8 | AR-43/11/18 | **`/api/v1` prefix on public surface only**; **short-TTL edge cache from V1** (s-maxage=60, credentialed ⇒ no-store); **keyset cursors in `query.Builder` in V1**, exposed on admin lists |

### Refinements made while writing this spec (deltas from the approved previews)

1. **`minRoleOwn` replaces the flat `predicate` key** in grant objects. The personas require role-dependent restriction (contributor edits own records; editor edits any — P-3/P-4, UAC-1.2), which a single `predicate` key applying uniformly to all granted roles cannot express. `Decision.Predicate` and its closed enum (`ownerOnly`) survive unchanged as the compilation target inside `query.Builder` (BR-RBAC-6 untouched); only the rule-JSON shape changes.
2. **API-key access is defined solely by the key's scopes**, not by intersection with the collection matrix. The matrix has no natural `api_key` entry, and both artifacts are provisioned by the same super admin — a second gate adds confusion, not security. Scope shape: `{"collections": [{"id": "<uuid>", "actions": ["read","create","update","delete"], "draftAccess": false}], "passwordReset": false}`; `draftAccess` implements BR-API-2's "unless the key scope explicitly grants draft access", and `passwordReset` gates the §2.3 request endpoint (a global capability, so it lives beside — not inside — the per-collection list).
3. **`super_admin` and `admin` hold an implicit full grant on content actions**; the matrix governs `editor`, `contributor`, `viewer`, `end_user`, and `anonymous`. Without this, a freshly created collection (empty rules + default-deny) would be invisible even to the admin who created it. This matches P-1 ("full system access") and P-2 ("manages content and schema"). BR-RBAC-3's default-deny is restated as applying to the governed classes.

---

## Section 1 — Access-Control Specification (D-1, resolves AR-1)

**New document `docs/architecture/12-access-rules.md`** containing:

### 1.1 Grant-matrix schema

`cms_collections.access_rules` is a JSONB object keyed by action (`read`, `create`, `update`, `delete`, `publish`). Each value is a grant object with **only** these keys, all optional:

| Key | Type | Meaning |
|---|---|---|
| `minRole` | role name | Admin principals with role ≥ this get the action **unrestricted** |
| `minRoleOwn` | role name | Admin principals with role ≥ this (and < `minRole` if both set) get the action **with `ownerOnly`** (`created_by = principal.ID`). Must be ≤ `minRole` in the lattice |
| `endUsers` | `"own"` \| `"all"` | End-user principals: `all` = unrestricted (within ScopePublic on reads), `own` = `ownerOnly`. Omitted = deny |
| `anonymous` | bool | Valid on `read` only. `true` = anonymous read under ScopePublic |

Role lattice: `viewer < contributor < editor < admin < super_admin`. Omitted action ⇒ deny for all governed classes (BR-RBAC-3). `publish` additionally floors at `editor` regardless of rules (BR-LIFE-3 unchanged). Unpublish evaluates as the `publish` action.

### 1.2 Worked examples (normative)

```jsonc
// Public blog: world-readable, editors run it
{ "read":    { "minRole": "viewer", "anonymous": true, "endUsers": "all" },
  "create":  { "minRole": "editor" },
  "update":  { "minRole": "editor" },
  "delete":  { "minRole": "editor" },
  "publish": { "minRole": "editor" } }

// Comments: end users own their rows; contributors moderate their own; editors moderate all
{ "read":    { "minRole": "viewer", "anonymous": true, "endUsers": "all" },
  "create":  { "endUsers": "all", "minRole": "contributor" },
  "update":  { "endUsers": "own", "minRoleOwn": "contributor", "minRole": "editor" },
  "delete":  { "minRole": "editor", "endUsers": "own" } }

// Ingestion target: no browser access at all; a write-scoped API key feeds it
{ "read":    { "minRole": "viewer" },
  "create":  {},              // admins implicit; key scope grants the rest
  "update":  {},
  "delete":  { "minRole": "admin" } }
```

### 1.3 Evaluation algorithm (prose, normative)

1. `super_admin`/`admin` → `Decision{Allowed: true}` for all content actions (Refinement 3); field rules still apply below `super_admin`.
2. Admin roles: `role ≥ minRole` → allowed unrestricted; else `role ≥ minRoleOwn` → allowed with `Predicate: ownerOnly`; else deny.
3. `end_user`: per `endUsers` (`own` ⇒ `ownerOnly`); reads additionally scoped by ScopePublic (BR-API-2).
4. `anonymous`: `read` + `anonymous: true` only, always ScopePublic.
5. `api_key`: allowed iff the key's scopes contain `(collection, action)`; reads published-only unless `draftAccess` (Refinement 2). Revoked keys resolve to `anonymous`.
6. Field rules resolve per §1.5 and ride along in `Decision.FieldRules`.

### 1.4 Validation (write-time, `httpapi/admin` + evaluator re-check)

Unknown action keys, unknown grant keys, unknown role names, `minRoleOwn > minRole`, `anonymous` outside `read`, or non-boolean/non-enum values ⇒ `422 validation_failed` naming the offending path. **Fail-closed at read:** rules that fail to parse at evaluation time deny (N-11).

### 1.5 Field-rule audiences (resolves the roles-only hole)

`hideFromRoles`/`readOnlyForRoles` in field `config` are **renamed** `hideFrom`/`readOnlyFor`, each a list drawn from the closed audience set: the five roles plus `end_user`, `anonymous`, `api_key`. Semantics unchanged (BR-RBAC-4): `hideFrom` strips on serialization, `readOnlyFor` rejects at `Document.Set`.

### 1.6 New business rule

- **BR-RBAC-7.** Access-rule objects conform to the closed grant-matrix schema of `12-access-rules.md`; writes that violate it are rejected with 422, and rules that fail validation at evaluation time deny. *Enforcement:* `httpapi/admin` rule validator + `access.Evaluator` fail-closed branch.

---

## Section 2 — Auth & Recovery (D-2, D-3, D-4; resolves AR-4, AR-5, AR-16, AR-17)

### 2.1 Super-admin recovery mode — new **BR-AUTH-12**

When `CMS_RECOVERY_EMAIL` is set at startup **and** names an existing `cms_users` row, the system generates a 256-bit single-use recovery token, logs it once at `warn`, and enables `/recover`, which accepts the token exactly once to set a new password for that user (and revokes that user's sessions). The token dies on use or process exit; the route returns 404 otherwise. Unset variable = feature absent. Mirrors BR-AUTH-11's pattern; same log-exception treatment (§8.3).

### 2.2 Admin-issued resets

A super admin (or admin, for non-super-admin targets per P-2 limits) generates a one-time reset token from the users screen; the token is displayed once (BR-AUTH-7 style), conveyed out-of-band, and consumed at a reset screen. Uses the same `cms_reset_tokens` mechanics as §2.3. Consuming an admin-issued reset token revokes all of the target user's sessions.

### 2.3 End-user password reset API — new **BR-AUTH-13**

- `POST /api/v1/auth/password-reset/request {email}` — requires an API key whose scopes carry `"passwordReset": true` (Refinement 2). Returns `{resetToken, expiresAt}` **to the consuming app**, which delivers it to the user through its own channel; unknown email returns `404 not_found` — the caller is a trusted server holding a key, so enumeration protection does not apply here (it applies on the public endpoints: `confirm`, `login`, `register`).
- `POST /api/v1/auth/password-reset/confirm {token, newPassword}` — public. On success: password updated, **all refresh-token families for that user revoked**, token marked used.
- **BR-AUTH-13.** Reset tokens store hashed, expire in 30 minutes, are single-use, and confirmation revokes every refresh-token family of the affected user. The binary never sends email; delivery is the consuming application's responsibility. *Enforcement:* `auth.ResetService` + `cms_reset_tokens`.

New table **`cms_reset_tokens`**: `id`, `user_kind` (`admin`|`end_user`), `user_id`, `token_hash` (unique), `expires_at`, `used_at`, `created_by`, `created_at`. Expired/used rows purged by `jobs.Retention` (§4.3).

### 2.4 Key custody — amend **BR-AUTH-10**

- `CMS_SESSION_SECRET` is **renamed `CMS_MASTER_SECRET`** (required). Its defined role: root secret for at-rest encryption of system key material. It plays no role in session tokens (which remain random values verified by hashed lookup — the AR-17 ambiguity is resolved by this sentence landing in 05).
- `cms_system_keys.private_pem` stores AES-256-GCM ciphertext under a key derived from `CMS_MASTER_SECRET` (HKDF); `public_pem` remains plaintext.
- Issued JWTs carry a `kid` header naming the signing key row; verification selects by `kid`, enabling overlap-window rotation.
- Rotation runbook (§7.4): generate successor keypair with new `kid` → issue with new key while verifying with both → retire old row after 15 min (max JWT TTL). Master-secret loss recovery: regenerate keypair; outstanding access JWTs (≤15 min) fail verification and clients silently re-issue via refresh tokens, which are opaque hashed rows independent of the RSA key.

### 2.5 Proxy trust — replaces 05 §5 heuristic (resolves AR-4, EC-10)

New env **`CMS_TRUSTED_PROXY_CIDRS`** (comma-separated). Resolution rule: if the direct peer is loopback, RFC1918, **or within a listed CIDR**, take the rightmost `X-Forwarded-For` entry that is *not* itself trusted; otherwise use the socket address and ignore XFF. Empty variable preserves today's behavior exactly. 09's proxy contract documents the Cloudflare-ranges one-liner and the failure mode (unlisted proxy ⇒ per-proxy-IP limiting — fails closed). If no untrusted XFF entry exists (header absent, or every entry trusted), the limiter uses the socket address.

### 2.6 Auth hygiene bundle (resolves AR-8, AR-24, AR-25)

Landing in 05 as normative text: session ID and CSRF token rotate on login (fixation); `register` and `refresh` rate-limit with BR-AUTH-6's existing numbers (10/15 min per email, 30/15 min per IP); login/register return uniform errors and always perform one Argon2id verification even for unknown emails (enumeration + timing); password policy is length-only — minimum 10 characters for admins, 8 for end users, no composition rules; the per-email limiter's targeted-lockout trade-off is documented explicitly. Argon2id parameters pinned: 64 MiB memory, 3 iterations, parallelism 2 (test-traceable constants).

### Env-variable table after this section (BUSINESS_RULES § Naming Constants)

14 → **16** rows: rename `CMS_SESSION_SECRET` → `CMS_MASTER_SECRET`; add `CMS_RECOVERY_EMAIL` (default: unset), `CMS_TRUSTED_PROXY_CIDRS` (default: empty).

---

## Section 3 — Correctness Contract Repairs (resolves AR-2, AR-3, AR-36, AR-46)

### 3.1 Optimistic locking (AR-2) — amend BR-LIFE-7 wording + 07

Normative sentence replacing the contradiction: *"Every save advances the live row's system columns (`version`, `updated_at`) via compare-and-set on `version`, including saves that create pending drafts; for published records the live row's **content columns** remain frozen until publish. The CAS is always against the live row's `version`."* Pending-draft detection and BR-LIFE-1/2 are unaffected; `cms_revisions.version_no` continues to equal the post-save `version`.

### 3.2 Locking narrative (AR-3) — amend 03 (EC-1 item 2, EC-12)

Corrections: `ALTER TABLE ... TYPE` and generated-column rebuilds take `ACCESS EXCLUSIVE` and **block reads as well as writes** for the rewrite's duration; the EC-12 sentence "reads proceed via MVCC" is deleted. Added ops guidance: expected stall ~seconds at the 100k-row design point, durations logged (08 already specifies), run heavy changes off-peak. The V2 FTS section gains the documented alternative if stalls become material: trigger-maintained tsvector column + `CREATE INDEX CONCURRENTLY` outside the schema transaction with a reconciliation step and a temporary staleness window.

### 3.3 Drain wording (AR-36) — amend N-6/BR-RUNTIME-6

*"Zero requests dropped among those completing within the 15-second window; requests exceeding it are force-closed, logged, and counted."*

### 3.4 Pagination error code (AR-46) — amend BR-API-1 + 04

Offset above 10,000 returns **`422 validation_failed`** (the registry has no 400; the registry stays closed).

---

## Section 4 — Single-Process & Reliability (resolves AR-9, AR-12, AR-13, AR-14, AR-15, AR-38, AR-39)

### 4.1 Single-instance enforcement — new **BR-RUNTIME-8**, new **EC-16**

Before opening the listener, the binary acquires a **process-lifetime advisory lock** (session-scoped `pg_advisory_lock` on a dedicated key, distinct from the migration and schema keys). A second process fails startup with a clear log line. This turns the design's deepest assumption — exactly one process — from convention into contract. EC-16 ("accidental second instance") enters the Edge-Case Register, resolved by 09. The migration-wait behavior in 09 is adjusted: the replacement process may wait on the *migration* lock, but startup fails if the *instance* lock is held (stop-then-start remains the only supported upgrade).

### 4.2 Health split (AR-12) — amend 08/09

`/healthz` = liveness (process up, always 200 once started; 503 during drain). New `/readyz` = readiness (DB ping). Proxies route on `/readyz`; supervisors restart on `/healthz`. A DB outage stops routing without triggering restart loops.

### 4.3 Jobs & growth (AR-13, AR-14, AR-15) — amend 07/08

Tickers wrap each tick in panic recovery; a panicking job logs at `error` and skips to the next tick, never killing the process. `jobs.Retention` gains duties 4–6: purge sessions expired per BR-AUTH-5, purge used/expired `cms_reset_tokens`, purge revoked/rotated refresh tokens older than 30 days. Rate-limiter maps are bounded LRU (fixed entry cap; eviction = forget oldest bucket).

### 4.4 Timeout table (AR-38) — new subsection in 09

`http.Server`: `ReadHeaderTimeout 5s`, `ReadTimeout 30s`, `WriteTimeout 30s`, `IdleTimeout 120s`. Per-request context deadline 25 s. Postgres `statement_timeout`: 10 s for collection queries, 60 s for schema transactions. Request-body caps: 5 MiB for record writes (must accommodate the 1 MiB per-field cap of §8.6 with headroom), 64 KiB for auth/presign/finalize and other non-content routes. (Values are normative defaults; constants, not env vars.)

### 4.5 Idempotent creates (AR-39) — amend 04

Public/API-key `POST` create endpoints accept optional `Idempotency-Key` (≤128 bytes). Same key + same principal within 24 h returns the original result instead of creating a duplicate. New table **`cms_idempotency_keys`**: `key_hash`, `principal_id`, `record_id`, `created_at`; unique on `(key_hash, principal_id)`; purged by retention after 24 h.

---

## Section 5 — Public API Contract (D-5, D-8; resolves AR-10, AR-11, AR-18, AR-43)

### 5.1 Filter/sort policy — new **BR-API-4**

`ScopePublic` accepts `filter`/`sort` only on fields whose config marks `indexed` or `unique`; violations return 422 naming the field and the remedy. `ScopeAdmin`/`ScopeTrash` accept any schema field (still subject to `Decision.FieldRules` visibility and `statement_timeout`). `query.Builder` invariant list (02 §query.Builder) gains this as invariant 7.

### 5.2 Counts opt-in — amend 04 envelope

`meta.pagination.total` appears only when `?count=exact` is requested; public consumers default to no count; the admin UI requests it where its tables need totals.

### 5.3 Versioning — amend 04 route map

Public surface moves under `/api/v1`: `/api/v1/collections/{slug}/records`, `/api/v1/auth/*`. Admin routes stay `/api/admin/*` (consumed only by the lockstep-shipped SPA). The envelope and error-code registry are documented as v1-stable and additive-only.

### 5.4 Edge cache — new **BR-API-5**, amend 04 + 09

Anonymous `ScopePublic` GETs: `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` + strong `ETag`. **Any request carrying `Authorization` or a cookie receives `Cache-Control: no-store` — this is the invariant (a cacheable credentialed response is a security bug).** `Vary: Authorization`. V2 purge-on-publish (webhooks) lengthens TTLs; V1 accepts ≤60 s publish propagation, documented. 09's proxy contract adds: the edge must honor origin cache headers on `/api/*` (already required for the SPA).

### 5.5 Keyset cursors in V1 — amend 02, 04, F-15/F-27

`query.Builder.Paginate` gains cursor support in V1 (opaque base64 of sort key + `id`; mutually exclusive with `offset` → 422). Admin list endpoints return `meta.pagination.next_cursor`; the admin UI pages next/prev beyond the 10k offset ceiling. F-27 becomes "expose cursors on the public API" (mechanism already built). Builder invariants extend: cursor pagination preserves the mandatory `id` tiebreaker and limit clamps.

---

## Section 6 — Media & Storage (resolves AR-19, AR-26)

- **MIME allowlist** (04): presign accepts a closed compile-time list in V1 (raster images, video, audio, PDF); admin-extensible configuration is deferred to V2 — no settings surface exists in V1 and this spec does not invent one. `Content-Type` is **signed into the presign policy** and re-verified against the HEAD response at finalize.
- **Bucket CORS** (09): required policy documented — `PUT` from the admin origin (and any registered app origins), no wildcard on credentialed requests. Called out as a UAC-1.5 prerequisite.
- **Origin isolation** (09): `R2_PUBLIC_BUCKET_URL` must not share a registrable domain with the admin UI origin (stored-HTML/phishing containment).
- **CSP fix** (06): `default-src 'self'; img-src 'self' <media domain>; media-src 'self' <media domain>` — media library previews work under the strict policy.
- **Media deletion** (07, new subsection): destructive-gated (BR-SCHEMA-7 style re-auth + confirmation); blocked with 409 while any `media` field references the record (FK RESTRICT); on success the row is deleted, then the object; the orphan sweep is the backstop for the crash window between the two.

---

## Section 7 — Ops & Compliance (D-6, D-7; resolves AR-6, AR-7, AR-20, AR-30, AR-42, AR-45)

1. **Backup/DR** (09 + REQUIREMENTS): PITR required — managed Postgres with PITR or self-hosted WAL archiving (pgBackRest/WAL-G) per runbook; scheduled `pg_dump` retained as the portable second copy. New **N-12: RPO ≤ 5 minutes, RTO ≤ 1 hour**, verified by a restore drill before V1 ships (runbook includes the drill).
2. **Availability statement** (REQUIREMENTS, new **N-13**): single-instance service; upgrades cost drain + startup (seconds); target availability class ~99.5%; HA is explicitly out of scope (cross-reference O-list).
3. **GDPR reserve** (REQUIREMENTS + 07): erasure semantics specified now — end-user hard delete (row + all refresh/reset tokens), `created_by` anonymized to a tombstone UUID, and a revision-redaction capability in `jobs.Retention` for content-embedded PII with documented limitations. Implemented as new V2 requirement **F-33**.
4. **Runbooks** (09, new section): JWT key rotation (§2.4), super-admin recovery (§2.1), restore drill, hot-DDL guidance (§3.2).
5. **Perf verification** (10 + REQUIREMENTS): new `make bench` target — seeded 100k-row database asserting N-3/N-4; added to the V1 delivery gate in 11.
6. **Audit durability note** (REQUIREMENTS, F-16): V1's audit trail is only as durable as shipped stdout logs; queryable persistence arrives in V2 (BR-AUDIT-2). Stated as an accepted limitation.

---

## Section 8 — Remaining Repairs (proposals folded in)

1. **DropCollection cascade** (03 + 07, amend BR-SCHEMA-7 addendum): dropping a collection deletes its `cms_fields`, `cms_revisions`, and table in one transaction; the typed-confirmation dialog states "destroys the table **and all revision history**"; the audit event records row/revision counts. (Field drop keeps revisions; collection drop does not — the asymmetry is now explicit.)
2. **Restore status** (07 + 06): restore returns the record with its prior `status`; the UI warns when restoring a published record that it becomes publicly visible immediately.
3. **Token-log exception** (BUSINESS_RULES hard-rule 6 + 08): setup (BR-AUTH-11) and recovery (BR-AUTH-12) tokens are the two sanctioned exceptions to "never log tokens" — single-use, `warn`-level, 30-minute TTL, die on use or exit. BR-AUTH-11 gains the TTL.
4. **Catch-up placement** (09 + BR-LIFE-9): the missed-schedule scan runs as the publisher's first tick immediately after the listener opens (readiness already true), bounding startup time; per-record late-publish `warn` logging unchanged. BR-LIFE-9's "on startup" wording adjusted accordingly.
5. **Schedule/time source** (07): `publish_at` comparisons use database `now()`, not process clock.
6. **Field-size cap** (02 `Document.Set` + 07): any single field value caps at 1 MiB (422 beyond); keeps revision snapshots within sizing assumptions.
7. **Composite indexes** (07): `indexed` fields get `(field, id)` B-trees, serving the mandatory tiebreaker sort directly.
8. **Batched expansion** (04): `?expand=` resolves via a single `IN` query per relation field, never per-row lookups.
9. **CI enforcement** (10): `depguard` (or equivalent) enforces the import rules table; added to `make test` prerequisites.
10. **OpenAPI** (new `docs/api/openapi.yaml`, referenced from 04): hand-maintained v1 public-surface spec; the filter grammar, envelope, error registry, and body caps get a machine-readable home.
11. **V2 scope pre-loads** (11 + 05 threat model): webhooks require a `cms_webhook_deliveries` outbox table (durable retry across restarts — Postgres-only, so BR-RUNTIME-2-compatible) and SSRF rules (deny private/link-local/loopback targets, re-resolve DNS at delivery, no redirects-to-private). Threat model §6 gains: SSRF, user enumeration, targeted rate-limit lockout, malicious media hosting, DB-read key theft (now mitigated by §2.4).
12. **Admin deletion semantics** (07): deleting an admin cascades sessions and reset tokens; `created_by` references persist by design (UUID retained for audit); audit events store actor snapshots.
13. **Doc-artifact fixes**: F-2's "the prompt's field-type reference" → `07-data-model.md`; 11's "generation contract" phrase removed (AR-31); 08's "only unauthenticated non-SPA endpoint" corrected to name `/readyz`, `/api/v1/auth/*`, and anonymous public reads (AR-47).
14. **New UAC** (REQUIREMENTS): **UAC-1.6** — an API consumer requests a password reset for an end user, the app delivers the token, confirmation succeeds, and every refresh-token family for that user is revoked; separately, a recovery-mode start resets a locked-out super admin and the token dies on use.

---

## Identifier Delta Summary

**New rules:** BR-RBAC-7, BR-AUTH-12, BR-AUTH-13, BR-RUNTIME-8, BR-API-4, BR-API-5.
**Amended rules:** BR-AUTH-10 (encryption + `kid`), BR-AUTH-11 (TTL), BR-LIFE-7 (CAS wording), BR-LIFE-9 (first-tick catch-up), BR-API-1 (422), BR-RUNTIME-6/N-6 (drain wording), BR-SCHEMA-7 (collection-drop asymmetry note), hard-rule 6 (token-log exception).
**New requirements:** F-32 (V1 recovery & reset), F-33 (V2 erasure), N-12 (RPO/RTO), N-13 (availability), UAC-1.6; amended F-15/F-27 (cursors), F-16 (audit note), F-2 (reference fix).
**New edge case:** EC-16 (accidental second instance → 09).
**New docs:** `docs/architecture/12-access-rules.md`, `docs/api/openapi.yaml` (skeleton). `00-README` document map and reading order updated.
**New tables (layouts reserved in 07):** `cms_reset_tokens`, `cms_idempotency_keys`; V2: `cms_webhook_deliveries`.
**Env vars (16):** rename `CMS_SESSION_SECRET`→`CMS_MASTER_SECRET`; add `CMS_RECOVERY_EMAIL`, `CMS_TRUSTED_PROXY_CIDRS`.
**New routes:** `/recover` (admin, gated), `/api/v1/auth/password-reset/{request,confirm}`, `/readyz`; public surface re-rooted under `/api/v1`.

## Per-Document Edit Matrix

| Document | Edits (sections above) |
|---|---|
| `BUSINESS_RULES.md` | New/amended rules per delta summary; env table 16 rows; hard-rule 6 exception; edge-case table + EC-16 |
| `REQUIREMENTS.md` | F-32, F-33, N-12, N-13, UAC-1.6; F-2/F-15/F-16/F-27 amendments; audit + availability notes |
| `00-README.md` | Add doc 12 (+ openapi.yaml) to map and reading order |
| `01-system-overview.md` | `/api/v1` in flows; EC-16 in register; readyz in lifecycle summary |
| `02-core-interfaces.md` | Builder invariant 7 (public indexed-only) + cursor support; `Document.Set` field-size cap; auth services `kid`/ResetService; Evaluator section points to doc 12 |
| `03-dynamic-schema.md` | §3.2 locking corrections; DropCollection cascade; FTS CONCURRENTLY alternative |
| `04-api-layer.md` | Route map v1; filter policy; count opt-in; cursors; cache headers; Idempotency-Key; MIME allowlist + presign Content-Type; reset routes; 422 for offset cap |
| `05-auth-security.md` | §5 replaced (CIDR rule); recovery mode; reset API; master secret + key encryption + `kid`; hygiene bundle (§2.6); threat-model additions |
| `06-admin-ui.md` | CSP img/media-src; reset-token affordance; `/recover` screen; cursor paging; restore-published warning; rule-matrix editor per doc 12 |
| `07-data-model.md` | CAS wording; new tables; composite indexes; media deletion; retention duties 4–6; erasure semantics; admin-deletion semantics; DB-time rule; field cap |
| `08-observability.md` | healthz/readyz split; token-log exceptions; job-telemetry additions |
| `09-deployment.md` | Startup order (instance lock; catch-up moved); timeout table; PITR + runbooks; CORS + origin isolation; proxy CIDR guidance; availability statement |
| `10-project-structure.md` | `depguard`; `make bench`; `docs/api/` in layout |
| `11-roadmap.md` | V1 scope additions; V2 additions (erasure, outbox, SSRF, purge-on-publish); bench in V1 gate; artifact-phrase fix |
| `12-access-rules.md` | **New** — Section 1 in full |

## Acceptance Criteria (for the remediation itself)

1. Every identifier in the delta summary exists in its named document; no document still references `CMS_SESSION_SECRET` (grep = 0).
2. The env table has exactly 16 rows and matches 09's configuration section.
3. `docs/architecture/12-access-rules.md` exists with the schema, algorithm, validation rules, and all three worked examples; 02, 05, and 06 reference it.
4. `grep -ri "proceed via MVCC" docs/` returns nothing; 03 states that rewrites block reads.
5. Every public route in 04 carries `/api/v1`; admin routes carry none.
6. `grep -ri "the prompt" docs/BUSINESS_RULES.md docs/REQUIREMENTS.md docs/architecture/` returns nothing.
7. AR-1…AR-5 each map to a completed section here (traceable in the review's terms); the review document gains a resolution-status appendix pointing at this spec.
8. BR-LIFE-7's new wording and 07's live-table contract no longer contradict (the "does not change" sentence is gone).
