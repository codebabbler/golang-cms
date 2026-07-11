# 05 — Auth & Security

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

Three principal types, three different threat models, three different mechanisms. This document incorporates the auth specification in full; the corresponding invariants live in `../BUSINESS_RULES.md` §4–§5 and tests trace to them.

| Principal | How it authenticates | Where it's used |
|---|---|---|
| **Admin user** | Email + password → session cookie + CSRF token | Admin UI, `/api/admin/*` |
| **API key** | Opaque bearer token | Server-to-server consumers |
| **Public JWT** | RS256, short-lived, refresh tokens | Browser/mobile end-user clients |

## 1. Admin Sessions

- **Login:** returns 200 with `Set-Cookie: cms_session=<256-bit-random>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`; the body includes `csrfToken` (BR-AUTH-1). Both the session ID and the CSRF token rotate on every successful login — a session or CSRF value established before authentication is discarded, defeating fixation.
- The cookie value is the lookup key. The database stores only the hash: `(token_hash, user_id, created_at, last_seen_at, ip, user_agent)` in `cms_sessions` (BR-AUTH-2). Session theft via database read yields nothing replayable. Session tokens are random 256-bit values verified by hashed lookup; no signing is involved. `CMS_MASTER_SECRET` is the root secret for at-rest encryption of system key material (§3) and plays no role in sessions.
- **Password hashing:** Argon2id with per-hash salts (BR-AUTH-3), parameters pinned at 64 MiB memory, 3 iterations, parallelism 2. These parameters apply to all principal kinds. Password policy is length-only — no composition rules: minimum 10 characters for admins, 8 characters for end users.
- **CSRF:** every state-changing admin request carries `X-CSRF-Token`, validated against the session's `csrf_hash` (BR-AUTH-4). The SPA holds the token in memory only (`06-admin-ui.md`).
- **Expiry:** 7 days idle, 30 days absolute. Destructive operations (schema drops, purges, key revocation) require re-authentication within the preceding 4 hours — `middleware.RequireRecentAuth` (BR-AUTH-5).
- **Rate limiting:** 10 attempts/15 min per email, 30 attempts/15 min per IP (BR-AUTH-6), evaluated before Argon2id work (`04-api-layer.md` middleware order).
- **First-Admin Bootstrap (BR-AUTH-11):** when `cms_users` is empty at startup, `auth.Bootstrap` generates a 256-bit random setup token, logs it exactly once at `warn`, and holds it in memory only — nothing persists. The admin UI's `/setup` screen accepts it exactly once to create the first super admin; the route returns 404 whenever `cms_users` is non-empty, and the token dies on use, after 30 minutes, or on process exit.

### Recovery Mode (BR-AUTH-12)

When `CMS_RECOVERY_EMAIL` is set at startup **and** names an existing `cms_users` row, `auth.Recovery` generates a 256-bit single-use recovery token, logs it once at `warn`, and enables `/recover`. The route accepts the token exactly once to set a new password for that user and revokes all of that user's sessions. The token dies on use, after 30 minutes, or on process exit; `/recover` returns 404 whenever recovery mode is not active. An unset `CMS_RECOVERY_EMAIL` means the feature is entirely absent — no route, no token, no log line. This mirrors BR-AUTH-11's bootstrap pattern and shares its log-exception treatment (`08-observability.md`). A `CMS_RECOVERY_EMAIL` naming no existing user logs one `warn` line and enables nothing.

**Admin-issued resets:** a super admin can generate a one-time reset token for any admin from the users screen; an admin can do the same only for non-super-admin targets (P-2 limits). The token displays exactly once (BR-AUTH-7 style), is conveyed out-of-band by the operator, and is consumed at a reset screen. This reuses the `cms_reset_tokens` mechanics described under Password Reset (§3) — same table, same hashing, same single-use/expiry discipline. Consuming an admin-issued reset token revokes all of the target user's sessions, exactly as recovery mode does.

## 2. API Keys (Server-to-Server)

- **Creation:** the plaintext token displays exactly once; the database stores `sha256(token)` in `cms_api_keys` (BR-AUTH-7).
- **Use:** `Authorization: Bearer cms_...`. The prefix makes leaked keys greppable in logs and repositories.
- **Scope:** per-collection scopes enforced at the collections handler through `access.Evaluator` — a key is a Principal with `Scopes`, never a superuser.
- **Revocation:** sets `revoked_at`; the row persists for audit. No auto-rotation.

## 3. Public JWTs (Client-Side End-Users)

For when the CMS acts as the user store (`cms_end_users`) for a client application.

- **Issuance:** RS256 JWT with 15-minute TTL plus an opaque refresh token. The JWT carries identity, NOT permissions; `access.Evaluator` resolves permissions server-side on every request (BR-AUTH-8), so a stale token never grants stale rights.
- **Algorithm rationale:** RS256 offers the broadest client-library compatibility; the system revisits EdDSA only if a concrete constraint appears.
- **Key management:** the RSA-2048 keypair persists in `cms_system_keys`, auto-generated at startup when `JWT_PRIVATE_KEY` is absent (BR-AUTH-10). The stored `private_pem` is AES-256-GCM ciphertext under a key derived from `CMS_MASTER_SECRET` via HKDF; `public_pem` remains plaintext. Every issued JWT carries a `kid` header naming the signing key row, so verification selects the matching key and rotation can run key overlap without invalidating live tokens.
- **Key rotation:** generate a successor keypair with a new `kid`; issue with the new key while still verifying against both `kid`s; retire the old row after 15 minutes (the maximum JWT TTL). If `CMS_MASTER_SECRET` is lost, recovery is to regenerate the keypair: outstanding access JWTs (≤15 minutes old) fail verification, and clients silently re-issue via their refresh tokens — which are opaque hashed rows independent of the RSA key, so no re-login is required.
- **Refresh:** refresh tokens store hashed in `cms_refresh_tokens`, grouped by `family_id`. Refreshing rotates both tokens.
- **Reuse detection:** presenting an already-rotated refresh token is theft evidence — the handler revokes the entire `family_id` and returns 401 (`unauthorized`); the legitimate client re-authenticates. *(Resolves EC-8; BR-AUTH-9.)*

### Password Reset (BR-AUTH-13)

- `POST /api/v1/auth/password-reset/request {email}` — requires an API key whose scopes carry `"passwordReset": true` (`12-access-rules.md` §1). Returns `{resetToken, expiresAt}` to the consuming app, which delivers it to the user through its own channel. An unknown email returns `404 not_found`: the caller is a trusted server holding a key, so the enumeration protection described in §5 does not apply here — it applies only to the public endpoints (`confirm`, `login`, `register`).
- `POST /api/v1/auth/password-reset/confirm {token, newPassword}` — public. On success: the password updates, **every refresh-token family for that user is revoked**, and the token is marked used.
- Reset tokens store hashed in `cms_reset_tokens`, expire in 30 minutes, and are single-use. The binary never sends email; delivery is the consuming application's responsibility.

## 4. RBAC

- **Roles:** `super_admin`, `admin`, `editor`, `contributor`, `viewer` — exactly five, CHECK-constrained (BR-RBAC-1).
- **Per-collection access rules:** JSONB config for `read`, `create`, `update`, `delete`, `publish` on `cms_collections`, conforming to the closed grant-matrix schema defined in `12-access-rules.md` (BR-RBAC-2, BR-RBAC-7). `access.Evaluator.Decide` returns `Decision{Allowed, Predicate, FieldRules}`; predicates like `ownerOnly` compile into every query (BR-RBAC-6). `super_admin` and `admin` hold an implicit full grant on content actions; the matrix itself governs `editor`, `contributor`, `viewer`, `end_user`, and `anonymous`.
- **Publish** is a distinct action that additionally floors at editor or above regardless of rules (BR-LIFE-3) — the persona line "Contributor cannot publish" is a Decision denial, not a UI convention.
- **Field-level access:** `hideFrom` strips fields from responses; `readOnlyFor` rejects writes at `Document.Set` (BR-RBAC-4). Audience lists draw from the closed set: the five roles plus `end_user`, `anonymous`, `api_key`.
- **Default-deny:** a missing rule denies for the governed classes (BR-RBAC-3). No allow-by-omission path exists outside the `super_admin`/`admin` implicit grant.

## 5. Client IP and Proxy Trust (EC-10)

Rate limiting keys on client IP, and the binary deploys behind an edge proxy (`09-deployment.md`) — so IP resolution must resist spoofing:

- If the direct socket peer is a loopback address, an RFC1918 private address, or falls within a CIDR listed in `CMS_TRUSTED_PROXY_CIDRS`, the limiter uses the **rightmost entry of `X-Forwarded-For` that is not itself trusted** — the address the outermost trusted proxy appended, immune to client-supplied prefix entries. If no untrusted entry exists (XFF absent, or every entry trusted), the limiter uses the socket address.
- Otherwise (the direct peer is untrusted) the limiter uses the socket address and **ignores** `X-Forwarded-For` entirely.
- An empty `CMS_TRUSTED_PROXY_CIDRS` preserves prior behavior exactly (loopback/RFC1918 trust only) — the variable is additive, never required.
- The deployment contract requires trusted proxies to append to (never blindly forward) `X-Forwarded-For`; `09-deployment.md` states this as a hard requirement and documents the Cloudflare-ranges configuration for `CMS_TRUSTED_PROXY_CIDRS`.

This rule is deterministic and fails safe: an unlisted proxy degrades to per-proxy-IP limiting rather than unlimited requests; a listed proxy that blind-forwards `X-Forwarded-For` is guarded by the append requirement above, not by this fallback. *(Resolves EC-10.)*

**Rate limiting extensions and enumeration posture:** `register` and `refresh` adopt BR-AUTH-6's existing numbers — 10 attempts/15 min per email (per end user, for `refresh`), 30 attempts/15 min per IP. `login` and `register` return uniform errors and always perform one Argon2id verification, even against an unknown email, so neither response shape nor timing discloses account existence. The per-email limiter carries a deliberate trade-off: an attacker who knows a victim's email can lock out that victim's own login attempts for the rate-limit window (targeted lockout); this is accepted because the alternative — no per-email limit — permits unbounded per-account credential brute force, which is the worse outcome.

## 6. Threat Model Summary

- CSRF mitigated by CSRF token + HttpOnly cookie.
- XSS mitigated by strict CSP and JSON escaping.
- Session token theft mitigated by DB hashing.
- SQL injection mitigated by pgx parameterized queries and strict schema identifiers (BR-SCHEMA-3).
- Brute force mitigated by Argon2id and rate limits.
- Mass assignment mitigated by schema-validated `Document.Set` (BR-RBAC-5).
- Refresh token replay mitigated by token-family revocation (BR-AUTH-9).
- IP spoofing against rate limits mitigated by the proxy-trust rule of §5.
- SSRF (V2 webhooks) mitigated by denying private/link-local/loopback delivery targets, re-resolving DNS at delivery time, and never following redirects into private address space.
- User enumeration mitigated by uniform errors on the public endpoints (`confirm`, `login`, `register`); constant Argon2id work on `login`/`register` (§5); `password-reset/request`'s trusted-caller exception is deliberate and scoped to API-key callers only (§3).
- Targeted rate-limit lockout is an accepted trade-off of the per-email limiter (§5), traded against unbounded per-account brute force.
- Malicious media hosting mitigated by the MIME allowlist enforced at presign (`04-api-layer.md`) and origin isolation between the media bucket and the admin UI origin (`09-deployment.md`).
- DB-read key theft mitigated by at-rest encryption of `cms_system_keys.private_pem` under `CMS_MASTER_SECRET` (§3, BR-AUTH-10) — a database read alone no longer yields a usable signing key.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-8 | Family revocation on rotated-token reuse (§3) |
| EC-10 | Deterministic peer-based `X-Forwarded-For` trust (§5) |
