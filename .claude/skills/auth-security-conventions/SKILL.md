---
name: auth-security-conventions
description: Use when touching internal/auth or any credential/token handling — password hashing, session issuance, API keys, JWTs, refresh rotation, or the first-admin bootstrap. Encodes the hashing discipline, token-family revocation, and key management rules.
---

# Auth Security Conventions

Distilled from `docs/architecture/05-auth-security.md`, `docs/BUSINESS_RULES.md` §4 (BR-AUTH-1…11), and `docs/architecture/07-data-model.md` (token tables). Those documents are authoritative.

**Boundary:** this skill owns auth *internals* — issuance, verification, storage. Route wiring and the normative middleware order belong to `api-conventions` — do not restate them here.

## The Hashing Discipline (memorize this table)

| Credential | At rest | Where |
|---|---|---|
| Admin/end-user passwords | Argon2id, per-hash salts (BR-AUTH-3) | `auth.Password` — the only hashing path |
| Session tokens | sha256 (BR-AUTH-2) | `cms_sessions.token_hash` |
| API keys | sha256 (BR-AUTH-7) | `cms_api_keys.token_hash` |
| Refresh tokens | sha256 (BR-AUTH-9) | `cms_refresh_tokens.token_hash` |

Raw token values never persist and never log, with two sanctioned exceptions: the single-use setup and recovery tokens (BR-AUTH-11/12), each logged once at `warn` with a 30-minute TTL (`docs/architecture/08-observability.md`). `CMS_MASTER_SECRET` encrypts system key material at rest (BR-AUTH-10); it plays no role in signing — session tokens are random 256-bit values verified by hashed lookup, never signed.

## Hard Rules

1. **Session cookie attributes are exact** (BR-AUTH-1): `cms_session=<256-bit-random>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`, body carries `csrfToken`. CSRF validates against `cms_sessions.csrf_hash` (BR-AUTH-4).
2. **Expiry windows** (BR-AUTH-5): 7 days idle, 30 days absolute, 4-hour recent-auth for destructive operations. These constants are business rules, not tunables.
3. **API keys** (BR-AUTH-7): plaintext displays exactly once at creation; the `cms_` prefix stays (leak-greppability); revocation sets `revoked_at` and the row persists for audit — never delete key rows.
4. **JWTs carry identity, never permissions** (BR-AUTH-8): RS256, 15-minute TTL; `access.Evaluator` resolves permissions per request. Adding a role or scope claim to the JWT is a defect even if it "saves a query."
5. **Key management** (BR-AUTH-10): the RSA-2048 keypair lives in `cms_system_keys`, auto-generated when `JWT_PRIVATE_KEY` is absent. RS256 stands until a concrete constraint justifies EdDSA.
6. **Refresh rotation + family revocation** (BR-AUTH-9, EC-8): every refresh rotates both tokens; presenting an already-rotated token revokes the entire `family_id` and returns 401. The family check precedes issuance — order matters.
7. **First-admin bootstrap** (BR-AUTH-11): when `cms_users` is empty at startup, `auth.Bootstrap` logs a 256-bit single-use setup token (once, at `warn`, memory-only); `/setup` consumes it to create the first super admin and returns 404 whenever `cms_users` is non-empty. The token dies on use or process exit. No bootstrap env vars exist — do not add any.
8. **Client IP resolution** (EC-10): private/loopback direct peer → rightmost non-private `X-Forwarded-For` entry; public direct peer → socket address, header ignored. Deterministic, no trusted-proxy configuration.

## Test Obligations

Trace to BR-AUTH-1…11. Highest-value adversarial tests: replaying a rotated refresh token kills the family; a database scan asserting no raw token substrings; `/setup` returning 404 with a non-empty `cms_users` even with a valid token; spoofed `X-Forwarded-For` from a public peer not moving the rate-limit key.

## Review Checklist

- [ ] Every stored credential appears in the hashing table with the right algorithm?
- [ ] No raw token in any persistence or log path?
- [ ] JWT claims still identity-only?
- [ ] Family check before refresh issuance?
- [ ] Bootstrap route dead on non-empty `cms_users`?
- [ ] `make trace` green for touched BR-AUTH rules?
