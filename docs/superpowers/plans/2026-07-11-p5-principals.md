# P5 Principals Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The two machine-facing principal kinds and their lifecycles — API keys with scopes, end-user JWTs with refresh rotation and reuse detection, the registration gate, password reset, and admin end-user management — plus the unified `middleware.Auth` that resolves all four principal kinds.

**Architecture:** Implements `05-auth-security.md` §2–§3 and the scope shape of `12-access-rules.md` §6; BR-AUTH-7…10, 13, 14. Key material lives in `cms_system_keys`, private PEM AES-256-GCM-encrypted under an HKDF key from `CMS_MASTER_SECRET` (BR-AUTH-10). `app.Run` gains the key-load startup step (failure aborts — N-11).

**Tech Stack:** github.com/golang-jwt/jwt/v5 (new dep), golang.org/x/crypto/hkdf, crypto/rsa + crypto/aes stdlib, sqlc over `cms_api_keys`/`cms_end_users`/`cms_refresh_tokens`/`cms_reset_tokens`/`cms_system_keys`.

> **Authored before execution (D-V1-3 amendment):** re-validate at execution start; docs win. Invoke `auth-security-conventions`, `sqlc-workflow`, and `api-conventions` before their tasks.

## Global Constraints

- API keys: plaintext shown exactly once, prefixed `cms_`, stored as `sha256(token)` hex (BR-AUTH-7); revocation sets `revoked_at`, row persists; a revoked key resolves to `anonymous` (12 §6) — no transport-layer error.
- Scope shape verbatim (12 §6): `{"collections":[{"id":"<uuid>","actions":["read","create","update","delete"],"draftAccess":false}],"passwordReset":false}` — `id` never slug; no `publish` action exists in scopes; validated on create with `422` naming the offending path.
- JWTs: RS256, **15-minute TTL**, identity-only claims (`sub` = end-user UUID, `iat`, `exp`, `kid` header) — never permissions (BR-AUTH-8). Verification selects by `kid` and accepts every live key row; issuance uses the newest (BR-AUTH-10 rotation overlap).
- Key at rest: RSA-2048 keypair per row in `cms_system_keys`; `private_pem` is AES-256-GCM ciphertext (nonce‖ciphertext, base64) under `HKDF-SHA256(CMS_MASTER_SECRET, info="cms_system_keys")`; `public_pem` plaintext. Auto-generated at startup when `JWT_PRIVATE_KEY` is absent; when present, that PEM is imported as the initial row. Key-load failure aborts startup (09).
- Refresh tokens: 256-bit opaque, sha256 at rest, `family_id` groups rotations; refresh rotates BOTH tokens; presenting an already-rotated token revokes the whole family and returns the uniform `401` (BR-AUTH-9, EC-8).
- Registration gate: `CMS_END_USER_REGISTRATION=enabled` else `POST /api/v1/auth/register` → **404** (BR-AUTH-14). End-user passwords ≥ 8 chars. `login`/`register` uniform errors + always one Argon2id verification; rate limits: 10/15 min per email (refresh: per end user), 30/15 min per IP (05 §5).
- Disable (BR-AUTH-14): sets `disabled_at`, revokes every refresh family; disabled users' `login`/`refresh` → uniform 401; their principal resolves as `anonymous`.
- Password reset (BR-AUTH-13): `request` requires an API key with `passwordReset: true`; unknown email → `404 not_found` (trusted-caller exception, 05 §3); returns `{resetToken, expiresAt}` — token conveyed by the consuming app. `confirm` is public; success updates the password, **revokes every refresh family** (end-user kind) or **every session** (admin kind), marks the token used. Tokens hashed in `cms_reset_tokens`, 30-min TTL, single-use. The binary never sends email.
- Cookies are never accepted on `/api/v1` (BR-API-6): the public-tree Auth middleware reads `Authorization` only.
- Never log tokens (hard rule 6); the `cms_` prefix exists to make leaks greppable in CI log-assertion tests (P8).
- Waiver shrink: `BR-AUTH-7 8 9 10 13 14`.
- Commits plain; branch `main`; `WriteError` only; test naming per BR convention.

## File Structure

```
internal/auth/keystore.go            system-key load/generate/encrypt (BR-AUTH-10)
internal/auth/keystore_integration_test.go
internal/auth/jwt.go                 JWTService: Issue/Verify (BR-AUTH-8)
internal/auth/jwt_test.go
internal/auth/refresh.go             refresh issue/rotate/reuse-revoke (BR-AUTH-9)
internal/auth/refresh_integration_test.go
internal/auth/apikey.go              APIKeyService (BR-AUTH-7) + scope validation (12 §6)
internal/auth/apikey_integration_test.go
internal/auth/reset.go               ResetService (BR-AUTH-13)
internal/auth/reset_integration_test.go
internal/httpapi/public_auth.go      /api/v1/auth/* handlers
internal/httpapi/public_auth_integration_test.go
internal/httpapi/auth_middleware.go  (modify) unified 4-kind resolution
internal/httpapi/admin_apikeys.go    /api/admin/api-keys
internal/httpapi/admin_endusers.go   /api/admin/end-users (BR-AUTH-14)
internal/httpapi/admin_users.go      /api/admin/users (list/create/role/admin-issued resets)
internal/store/queries/{apikeys,endusers,refresh,resets,systemkeys}.sql
internal/app/app.go                  (modify) key-load step; wire services
docs/trace-waivers.txt               (modify) shrink
```

---

### Task 1: Keystore — cms_system_keys under the master secret

**Files:**
- Create: `internal/auth/keystore.go`, `internal/store/queries/systemkeys.sql`
- Test: `internal/auth/keystore_integration_test.go`

**Interfaces:**
- sqlc: `InsertSystemKey(name, private_pem, public_pem, created_at)`, `ListSystemKeys`, `DeleteSystemKey(name)`.
- Produces:
  - `auth.NewKeystore(q *store.Queries, masterSecret string) *Keystore`.
  - `(*Keystore).Load(ctx, importPEM string) error` — startup step: decrypt+parse every row; zero rows → generate RSA-2048 (or import `importPEM` = `Config.JWTPrivateKey` when set), name it `jwt-<8hexrand>` (the `kid`), encrypt, insert. Any decrypt/parse failure → error (aborts startup, N-11 — a wrong `CMS_MASTER_SECRET` fails closed; the 09 runbook's regenerate path is `DeleteSystemKey` + restart, deliberately manual).
  - `(*Keystore).Signing() (kid string, key *rsa.PrivateKey)` — newest row. `(*Keystore).Public(kid string) (*rsa.PublicKey, bool)` — any live row.
  - Crypto helpers (package-private, unit-testable): `sealPEM(secret, plaintext string) string` / `openPEM(secret, sealed string) (string, error)` — AES-256-GCM, key = HKDF-SHA256(secret, salt=nil, info=`"cms_system_keys"`), random 12-byte nonce prepended, base64std.

- [ ] **Step 1: failing tests** — seal/open round-trip; open with wrong secret errors (`TestBR_AUTH_10_PrivateKeyUnreadableWithoutMasterSecret` — DB-read theft mitigation); integration: fresh DB `Load` generates one row whose `private_pem` does NOT contain `"BEGIN RSA"` (ciphertext check) and `public_pem` does; second `Load` reuses (no second row); `Load` with doctored ciphertext errors.
- [ ] **Step 2–4:** implement (`go get github.com/golang-jwt/jwt/v5@latest` here too), PASS. **Step 5: Commit** — `git commit -m "feat: encrypted system keystore with startup load (BR-AUTH-10)"`

---

### Task 2: JWTService

**Files:**
- Create: `internal/auth/jwt.go`
- Test: `internal/auth/jwt_test.go` (unit — keystore stubbed with a generated key)

**Interfaces:**
- Produces:
  - `auth.NewJWTService(ks *Keystore) *JWTService`; constant `AccessTokenTTL = 15 * time.Minute`.
  - `(*JWTService).Issue(endUserID uuid.UUID, now time.Time) (string, error)` — claims exactly `{sub, iat, exp}`, header `kid` = signing key's; RS256.
  - `(*JWTService).Verify(token string) (uuid.UUID, error)` — selects key by `kid`; rejects unknown kid, expired, wrong alg (alg confusion: only RS256 accepted), malformed sub.

- [ ] **Step 1: failing tests** — `TestBR_AUTH_8_ClaimsAreIdentityOnly` (decode payload, assert key set == {sub,iat,exp}); round-trip verify; expiry at +15m1s fails; `alg=HS256` token signed with the public PEM as HMAC key rejected (classic confusion attack); unknown `kid` rejected; second key added → old tokens still verify, new issues use new kid (`TestBR_AUTH_10_RotationOverlap`).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: RS256 identity-only JWTs with kid rotation (BR-AUTH-8/10)"`

---

### Task 3: Refresh tokens — rotation and reuse detection

**Files:**
- Create: `internal/auth/refresh.go`, `internal/store/queries/{endusers,refresh}.sql`
- Test: `internal/auth/refresh_integration_test.go`

**Interfaces:**
- sqlc: endusers — `CreateEndUser`, `GetEndUserByEmail`, `GetEndUserByID`, `UpdateEndUserPassword`, `SetEndUserDisabledAt`, `ListEndUsers` (paged); refresh — `InsertRefreshToken`, `GetRefreshTokenByHash`, `MarkRefreshRotated(id, rotated_at)`, `RevokeFamily(family_id, revoked_at)`, `RevokeAllFamiliesForEndUser(end_user_id, revoked_at)`.
- Produces:
  - `auth.NewRefreshService(q *store.Queries) *RefreshService`.
  - `(*RefreshService).Issue(ctx, endUserID uuid.UUID) (plaintext string, err error)` — new family (family_id = new UUIDv7).
  - `(*RefreshService).Rotate(ctx, plaintext string) (endUserID uuid.UUID, newPlaintext string, err error)` — lookup by hash: not found → `ErrRefreshInvalid`; `revoked_at` set → `ErrRefreshInvalid`; `rotated_at` set → **revoke the family** and return `ErrRefreshReused` (BR-AUTH-9; handler → uniform 401 either way — split errors exist for tests/audit only); live → mark rotated, insert successor in the same family, same tx.
  - `(*RefreshService).RevokeAllFor(ctx, endUserID) error` (disable/reset paths).
  - Disabled check: `Rotate` joins `cms_end_users.disabled_at` — non-null → `ErrRefreshInvalid` (BR-AUTH-14).

- [ ] **Step 1: failing tests** — rotate happy path (old dies, new works); `TestBR_AUTH_9_ReuseRevokesFamily`: issue → rotate → present the ORIGINAL again → ErrRefreshReused AND the rotated successor no longer works (family dead — EC-8); revoke-all kills every family; disabled user's rotate fails.
- [ ] **Step 2–4:** implement (+`make generate`), PASS. **Step 5: Commit** — `git commit -m "feat: refresh rotation with family revocation on reuse (BR-AUTH-9, EC-8)"`

---

### Task 4: Public auth routes — register, login, refresh, logout

**Files:**
- Create: `internal/httpapi/public_auth.go`
- Modify: `internal/httpapi/router.go` (mount `/api/v1/auth/*`; RateLimit slots per 05 §5), `internal/httpapi/ratelimit.go` (register/login/refresh buckets)
- Test: `internal/httpapi/public_auth_integration_test.go`

**Interfaces:**
- Consumes: `Hasher` + `DummyHash` (P2), `JWTService`, `RefreshService`, end-user queries, `Limiter`, `Config.EndUserRegistration`.
- Produces routes (all under the `/api/v1` CORS-carrying subtree from P6 — mounted now, CORS middleware added in P6):
  - `POST /api/v1/auth/register {email, password}` — gate: disabled → `404 not_found` (BR-AUTH-14); limits 10/15 m email + 30/15 m IP; password ≥ 8; duplicate email → uniform behavior: perform the dummy hash and return the SAME success-shaped `201 {data:{}}`? No — 05 says uniform errors on register; duplicate email returns the uniform `422 validation_failed` "registration failed" without naming the email as taken, after one Argon2id hash — enumeration-resistant. Success: create user (Argon2id) → `201` with `{data:{accessToken, refreshToken}}` (registration logs the user in — openapi's 201 "Account created.").
  - `POST /api/v1/auth/login {email, password}` — limits; unknown email → verify `DummyHash`; disabled → uniform; success → `{data:{accessToken, refreshToken}}` 200.
  - `POST /api/v1/auth/refresh {refreshToken}` — per-user limit keyed on the token's user after lookup (pre-lookup key: IP), 30/15 m IP; `Rotate` → `{data:{accessToken, refreshToken}}`; any failure → uniform `401 unauthorized`.
  - `POST /api/v1/auth/logout {refreshToken}` — revokes the token's family; always `200` (idempotent, no oracle).
- All bodies under the 64 KiB cap (P1 default).

- [ ] **Step 1: failing integration test** — gate off → register 404 (`TestBR_AUTH_14_RegistrationGateDefaultDisabled`); gate on → register 201 issues working tokens; login wrong password vs unknown email: byte-identical status+body (`TestUniformLoginErrors`); refresh rotation arc incl. reuse → 401 and family dead (HTTP-level BR-AUTH-9 rerun); 11th email-keyed login attempt → 429.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: end-user register/login/refresh/logout with uniform errors (BR-AUTH-9/14)"`

---

### Task 5: API keys and unified principal resolution

**Files:**
- Create: `internal/auth/apikey.go`, `internal/store/queries/apikeys.sql`, `internal/httpapi/admin_apikeys.go`
- Modify: `internal/httpapi/auth_middleware.go` (unify), `internal/httpapi/router.go`
- Test: `internal/auth/apikey_integration_test.go`, extend `auth_middleware_test.go`

**Interfaces:**
- sqlc: `InsertAPIKey`, `GetAPIKeyByHash`, `RevokeAPIKey(id, revoked_at)`, `ListAPIKeys`.
- Produces:
  - `access.CollectionScope{CollectionID uuid.UUID; Actions []string; DraftAccess bool}`; `access.Principal.Scopes []CollectionScope` + `PasswordReset bool` field on a new `access.KeyGrants` carried in Principal? Keep 02's shape: `Principal.Scopes []CollectionScope`; the global `passwordReset` capability rides as `Principal.PasswordReset bool` (02 names only Scopes — additive field, flagged for execution review).
  - `auth.NewAPIKeyService(q *store.Queries) *APIKeyService`; `Create(ctx, name string, scopes ScopeDoc) (plaintext string, id uuid.UUID, err error)` — plaintext `"cms_" + base64url(32 rand bytes)`; `Verify(ctx, bearer string) (access.Principal, error)` — sha256 lookup; revoked → returns anonymous principal, nil error (12 §6); `Revoke(ctx, id) error`.
  - `auth.ScopeDoc` mirrors the 12 §6 JSON; `ValidateScopeDoc(sd ScopeDoc) []content.FieldError`-style errors: unknown action, `publish` present, non-UUID id → `422` paths.
  - Unified `Auth` middleware: admin tree (`/api/admin`, `/setup`, `/recover`) — cookie → admin principal (P2 behavior unchanged); public tree (`/api/v1`) — `Authorization: Bearer cms_*` → APIKeyService.Verify; other `Bearer` → JWTService.Verify → end-user principal (disabled → anonymous); absent/invalid → anonymous. Cookies ignored on `/api/v1`.
  - Admin routes `/api/admin/api-keys`: `GET /` list (hashes never serialized), `POST /` create (plaintext returned ONCE — BR-AUTH-7; response includes the scope echo), `DELETE /{id}` revoke (RequireRecentAuth — key lifecycle is destructive per BR-AUTH-5's destructive set). Role floor: `admin`+.

- [ ] **Step 1: failing tests** — `TestBR_AUTH_7_PlaintextOnceHashAtRest` (row stores sha256, list response omits hashes, plaintext prefix `cms_`); revoked key resolves anonymous, row persists; scope validation rejects `publish` and slug-shaped ids; middleware kind-dispatch table (cookie on /api/v1 ignored → anonymous; `cms_` bearer → api_key; JWT → end_user).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: API keys with 12 §6 scopes; unified four-kind principal resolution (BR-AUTH-7)"`

---

### Task 6: ResetService and password-reset routes

**Files:**
- Create: `internal/auth/reset.go`, `internal/store/queries/resets.sql`
- Modify: `internal/httpapi/public_auth.go` (request/confirm), `internal/httpapi/admin_users.go` (create: admin-issued resets + user management)
- Test: `internal/auth/reset_integration_test.go`, extend `public_auth_integration_test.go`

**Interfaces:**
- sqlc: `InsertResetToken`, `GetResetTokenByHash`, `MarkResetUsed`, plus users-list queries for the admin screen (`ListUsers`).
- Produces:
  - `auth.NewResetService(q *store.Queries, hasher *Hasher, sessions *SessionService, refresh *RefreshService) *ResetService`.
  - `Request(ctx, kind string /* admin|end_user */, userID uuid.UUID, createdBy uuid.UUID) (plaintext string, expiresAt time.Time, err error)` — 256-bit, sha256 at rest, 30-min TTL.
  - `Confirm(ctx, plaintext, newPassword string) error` — single-use check + TTL; end_user kind → `RevokeAllFor` (BR-AUTH-13); admin kind → `DeleteSessionsForUser` (05 §1); password floors 10/8 by kind.
  - Routes: `POST /api/v1/auth/password-reset/request {email}` — requires api_key principal with `PasswordReset` capability → else `403 forbidden`; unknown email → `404` (trusted-caller exception); → `{data:{resetToken, expiresAt}}`. `POST /api/v1/auth/password-reset/confirm {token, newPassword}` — public, kind-dispatching (the token row knows its kind — admin-issued tokens confirm here too; planning decision, flagged: 05 says "a reset screen" without naming a route, and one public confirm endpoint serves both kinds with identical mechanics).
  - `/api/admin/users`: `GET /` list; `POST /` create admin user (role in body; super_admin creatable by super_admin only — P-2); `POST /{id}/reset-token` — admin-issued reset (super_admin: any target; admin: non-super targets only → else 403); response carries the plaintext once.
- [ ] **Step 1: failing tests** — `TestBR_AUTH_13_ConfirmRevokesRefreshFamilies` (end-user arc: request via passwordReset-scoped key → confirm → old refresh dead, new password logs in, token single-use); request without capability → 403; unknown email → 404; admin-issued arc revokes sessions; P-2 limit (admin cannot reset a super_admin).
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: password reset with family/session revocation; admin-issued resets (BR-AUTH-13)"`

---

### Task 7: Admin end-user management, wiring, smoke, waiver shrink

**Files:**
- Create: `internal/httpapi/admin_endusers.go`
- Modify: `internal/app/app.go` (keystore load between migrations and schema cache — 09 startup order; wire all services), `docs/trace-waivers.txt` (remove BR-AUTH-7,8,9,10,13,14)
- Test: `internal/httpapi/admin_endusers_integration_test.go`, extend `internal/app/smoke_integration_test.go`

**Interfaces:**
- Routes `/api/admin/end-users` (RequireSession + role `admin`+): `GET /` list (paged); `POST /{id}/disable` — `SetEndUserDisabledAt(now)` + `RevokeAllFor` (BR-AUTH-14); `POST /{id}/enable` — clears `disabled_at`; `POST /{id}/revoke-refresh` — families only.
- Smoke leg: gate on → register end user → refresh works → admin disables → refresh 401 + login 401 → enable → login works.

- [ ] **Steps:** failing integration test (`TestBR_AUTH_14_DisableRevokesAndResolvesAnonymous` — includes: disabled user's still-valid 15-min JWT resolves to anonymous on a protected read once P6 lands; in P5 assert via `PrincipalFromContext` middleware test), implement, wire startup (`Keystore.Load` failure aborts — add to the fail-closed test), `make test && make trace && make lint` green (waived count −6), commit — `git commit -m "feat: admin end-user management; P5 smoke green (BR-AUTH-14)"`.

## Plan Self-Review Notes

- Registration success returns tokens (auto-login) — inferred from openapi's `201 Account created.` + Envelope; if execution-time review prefers 201-without-tokens, both satisfy the docs; pick one and mirror it in the P9 client. Flagged.
- One public confirm endpoint serving both reset kinds is a planning decision (05 names no admin-confirm route); flagged for execution review.
- `Principal.PasswordReset bool` is an additive field beyond 02's schematic — minor per 02 §Stability Rules; flagged.
- JWT middleware resolves disabled users to anonymous by re-checking `disabled_at` per request (BR-AUTH-8's "permissions resolve server-side per request" makes the DB hit mandatory, not optional).
- Refresh per-user rate limit applies post-lookup (the user is unknown until then); pre-lookup abuse is bounded by the 30/15 m IP bucket — matches 05 §5's "per end user, for refresh".
