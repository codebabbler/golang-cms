---
name: api-conventions
description: Use when adding or changing any HTTP handler, route, or middleware in internal/httpapi — request parsing, responses, errors, pagination, or auth wiring. Encodes the envelope, the error-code registry, and the normative middleware order.
---

# API Conventions

Distilled from `docs/architecture/04-api-layer.md` and `docs/architecture/02-core-interfaces.md`. Those documents are authoritative.

## Middleware Order Is Normative

```text
RequestID → Logger → Recover → RateLimit → Auth
  admin:  → RequireSession → RequireCSRF (mutations) → RequireRecentAuth (destructive)
```

The order is a security surface: rate limiting precedes auth so credential stuffing burns the limiter, not Argon2id time; CSRF runs only after a valid session. Changing the order requires a BUSINESS_RULES review — do not reorder casually.

## Response Envelope

- Success: `{ "data": ..., "meta": { "pagination": ... } }`.
- Error: `{ "error": { "code", "message", "details" } }` — written **only** by `httpapi.WriteError` (BR-API-3). Never hand-roll an error JSON.
- Codes are a closed registry: `validation_failed` 422, `unauthorized` 401, `forbidden` 403, `not_found` 404, `conflict` 409, `rate_limited` 429, `payload_too_large` 413, `internal` 500. A new code is a documented API change to `04-api-layer.md`, not a local decision.
- `409 conflict` covers exactly: optimistic-lock mismatch (BR-LIFE-7), trash-restore collision (BR-LIFE-5), slug collisions.

## Handler Rules

1. Resolve `Decision` via `access.Evaluator.Decide` **before** touching `query.Builder`; the builder requires it for collection scopes (BR-RBAC-2).
2. All writes go through `content.Document.Set` — it is the only write path (BR-RBAC-5); field errors from it map to `validation_failed` details.
3. Pagination via the shared `httpapi.ParsePagination`: limit default 25 / max 100, offset ceiling 10,000 → 400, `id` tiebreaker always (BR-API-1, EC-11).
4. Filters/sorts pass field names straight to the builder — it validates against schema and visibility; handlers never pre-validate field existence themselves (single source of truth).
5. Relation expansion is one level (`?expand=`); public-scope targets that are draft/trashed/hidden serialize as `null` (BR-LIFE-6, EC-7).
6. Media: presign (size ≤ 50 MB V1 constant, 15-min expiry) → client PUTs directly → finalize verifies via storage HEAD. No route ever accepts file bytes (BR-MEDIA-1).
7. `richText` is Tiptap JSONB in and out (V1). Never store or emit HTML in V1 paths.
8. Log via the request-scoped `slog` logger; never log tokens, cookie values, presigned URLs, or JWT bodies (`docs/architecture/08-observability.md`).

## Review Checklist

- [ ] Route registered under the correct prefix with the correct middleware chain?
- [ ] Errors exclusively via `WriteError` with registry codes?
- [ ] `Decide` before builder; `Document.Set` for writes?
- [ ] Pagination through `ParsePagination`?
- [ ] Destructive route added to the `RequireRecentAuth` set if it deletes/drops/revokes?
- [ ] BR-traced tests; `make trace` green?
