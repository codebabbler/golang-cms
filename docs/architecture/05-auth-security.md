# 05 — Auth & Security

**Version:** 1.0 · **Last Updated:** 2026-07-08 · **Owner:** Miraj Aryal

Three principal types, three different threat models, three different mechanisms. This document incorporates the auth specification in full; the corresponding invariants live in `../BUSINESS_RULES.md` §4–§5 and tests trace to them.

| Principal | How it authenticates | Where it's used |
|---|---|---|
| **Admin user** | Email + password → session cookie + CSRF token | Admin UI, `/api/admin/*` |
| **API key** | Opaque bearer token | Server-to-server consumers |
| **Public JWT** | RS256, short-lived, refresh tokens | Browser/mobile end-user clients |

## 1. Admin Sessions

- **Login:** returns 200 with `Set-Cookie: cms_session=<256-bit-random>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`; the body includes `csrfToken` (BR-AUTH-1).
- The cookie value is the lookup key. The database stores only the hash: `(token_hash, user_id, created_at, last_seen_at, ip, user_agent)` in `cms_sessions` (BR-AUTH-2). Session theft via database read yields nothing replayable.
- **Password hashing:** Argon2id with per-hash salts (BR-AUTH-3). `CMS_SESSION_SECRET` signs cookies; it plays no role in password hashing.
- **CSRF:** every state-changing admin request carries `X-CSRF-Token`, validated against the session's `csrf_hash` (BR-AUTH-4). The SPA holds the token in memory only (`06-admin-ui.md`).
- **Expiry:** 7 days idle, 30 days absolute. Destructive operations (schema drops, purges, key revocation) require re-authentication within the preceding 4 hours — `middleware.RequireRecentAuth` (BR-AUTH-5).
- **Rate limiting:** 10 attempts/15 min per email, 30 attempts/15 min per IP (BR-AUTH-6), evaluated before Argon2id work (`04-api-layer.md` middleware order).
- **First-Admin Bootstrap (BR-AUTH-11):** when `cms_users` is empty at startup, `auth.Bootstrap` generates a 256-bit random setup token, logs it exactly once at `warn`, and holds it in memory only — nothing persists. The admin UI's `/setup` screen accepts it exactly once to create the first super admin; the route returns 404 whenever `cms_users` is non-empty, and the token dies on use or process exit.

## 2. API Keys (Server-to-Server)

- **Creation:** the plaintext token displays exactly once; the database stores `sha256(token)` in `cms_api_keys` (BR-AUTH-7).
- **Use:** `Authorization: Bearer cms_...`. The prefix makes leaked keys greppable in logs and repositories.
- **Scope:** per-collection scopes enforced at the collections handler through `access.Evaluator` — a key is a Principal with `Scopes`, never a superuser.
- **Revocation:** sets `revoked_at`; the row persists for audit. No auto-rotation.

## 3. Public JWTs (Client-Side End-Users)

For when the CMS acts as the user store (`cms_end_users`) for a client application.

- **Issuance:** RS256 JWT with 15-minute TTL plus an opaque refresh token. The JWT carries identity, NOT permissions; `access.Evaluator` resolves permissions server-side on every request (BR-AUTH-8), so a stale token never grants stale rights.
- **Algorithm rationale:** RS256 offers the broadest client-library compatibility; the system revisits EdDSA only if a concrete constraint appears.
- **Key management:** the RSA-2048 keypair persists in `cms_system_keys`, auto-generated at startup when `JWT_PRIVATE_KEY` is absent (BR-AUTH-10).
- **Refresh:** refresh tokens store hashed in `cms_refresh_tokens`, grouped by `family_id`. Refreshing rotates both tokens.
- **Reuse detection:** presenting an already-rotated refresh token is theft evidence — the handler revokes the entire `family_id` and returns 401 (`unauthorized`); the legitimate client re-authenticates. *(Resolves EC-8; BR-AUTH-9.)*

## 4. RBAC

- **Roles:** `super_admin`, `admin`, `editor`, `contributor`, `viewer` — exactly five, CHECK-constrained (BR-RBAC-1).
- **Per-collection access rules:** JSONB config for `read`, `create`, `update`, `delete` on `cms_collections`. `access.Evaluator.Decide` returns `Decision{Allowed, Predicate, FieldRules}`; predicates like `ownerOnly` compile into every query (BR-RBAC-2, BR-RBAC-6).
- **Publish** is a distinct action requiring editor or above (BR-LIFE-3) — the persona line "Contributor cannot publish" is a Decision denial, not a UI convention.
- **Field-level access:** `hideFromRoles` strips fields from responses; `readOnlyForRoles` rejects writes at `Document.Set` (BR-RBAC-4).
- **Default-deny:** a missing rule denies (BR-RBAC-3). No allow-by-omission path exists.

## 5. Client IP and Proxy Trust (EC-10)

Rate limiting keys on client IP, and the binary deploys behind an edge proxy (`09-deployment.md`) — so IP resolution must resist spoofing without adding configuration:

- If the direct socket peer is a loopback or RFC1918 private address (the reverse proxy), the limiter uses the **rightmost non-private** entry of `X-Forwarded-For` — the address the trusted proxy appended, immune to client-supplied prefix entries.
- If the direct peer is a public address (no proxy), the limiter uses the socket address and **ignores** `X-Forwarded-For` entirely.
- The deployment contract requires the reverse proxy to append to (never blindly forward) `X-Forwarded-For`; `09-deployment.md` states this as a hard requirement.

This rule is deterministic, needs no trusted-proxy list, and fails safe: a misconfigured proxy chain degrades to per-proxy-IP limiting rather than unlimited requests. *(Resolves EC-10.)*

## 6. Threat Model Summary

- CSRF mitigated by CSRF token + HttpOnly cookie.
- XSS mitigated by strict CSP and JSON escaping.
- Session token theft mitigated by DB hashing.
- SQL injection mitigated by pgx parameterized queries and strict schema identifiers (BR-SCHEMA-3).
- Brute force mitigated by Argon2id and rate limits.
- Mass assignment mitigated by schema-validated `Document.Set` (BR-RBAC-5).
- Refresh token replay mitigated by token-family revocation (BR-AUTH-9).
- IP spoofing against rate limits mitigated by the proxy-trust rule of §5.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-8 | Family revocation on rotated-token reuse (§3) |
| EC-10 | Deterministic peer-based `X-Forwarded-For` trust (§5) |
