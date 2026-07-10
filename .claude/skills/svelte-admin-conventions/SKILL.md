---
name: svelte-admin-conventions
description: Use when building or changing the admin UI in web/ — Svelte 5 components, the API client, Tiptap integration, CSRF handling, or editorial state displays. Encodes the SPA conventions the embedded UI must follow.
---

# Svelte Admin Conventions

Distilled from `docs/architecture/06-admin-ui.md`. That document is authoritative.

## Stack Rules

- Svelte 5 + Vite, **no SvelteKit** (O-5) — the build must stay embeddable static assets (`vite build` → `web/dist` → `go:embed`).
- State via runes (`$state`, `$derived`, `$effect`); shared state in module-scope rune stores; no external state library.
- No new runtime dependency without checking the CSP consequence: `default-src 'self'`, no inline scripts, everything bundles at build time.

## The API Client (one module, no exceptions)

All server access goes through the single client module. It owns:

- Envelope parsing (`data`/`meta` vs `error.code` → typed failures).
- `X-CSRF-Token` on every mutation. The token lives **in memory only** — never `localStorage`, never a readable cookie (BR-AUTH-4). Hard refresh refetches via `/api/admin/auth/csrf`.
- `401` → redirect to login. `403` with recent-auth detail → re-auth modal, then retry the original request (BR-AUTH-5).
- The session cookie is `HttpOnly` — the SPA never reads or writes it.

Components calling `fetch` directly are a defect.

## Editorial UI Rules

- Record header state is exactly one of: **Draft**, **Published**, **Published + pending draft** (newest revision beyond the published one — `docs/architecture/07-data-model.md`).
- Render publish/unpublish controls only when the server's `Decision` allows; Contributors see `ownerOnly` drafts and no publish control.
- `409` on save opens the conflict dialog: reload-and-reapply only, **no merge UI** (O-10 — no collaborative editing).
- Destructive flows (drop field/collection, purge): typed-slug confirmation client-side, knowing the server independently enforces it plus recent re-auth (BR-SCHEMA-7).
- Trash-restore `409` surfaces the colliding field with a link to the colliding record.
- Revision restore previews drift outcomes (dropped/defaulted fields) before submitting (EC-5).

## Tiptap Rules

- Tiptap exchanges **canonical JSONB** with the API; the editor never produces HTML for storage, and rich text renders through Tiptap's schema-constrained renderer — never `innerHTML` of stored strings.
- Type-change pickers in the schema builder offer only safe-matrix targets (`docs/architecture/03-dynamic-schema.md`).

## Media Upload Flow

Presign → direct PUT to storage with progress → finalize. Abandoned uploads need no client cleanup — the server orphan sweep owns them (EC-9).

## Review Checklist

- [ ] All server calls through the client module?
- [ ] CSRF token never persisted; no cookie reads?
- [ ] New views respect the three-state editorial model and Decision-gated controls?
- [ ] Rich text stays JSONB end-to-end?
- [ ] No CSP-breaking pattern (inline script, remote asset)?
