# 06 — Admin UI

**Version:** 1.0 · **Last Updated:** 2026-07-08 · **Owner:** Miraj Aryal

The admin UI is a Svelte 5 + Vite SPA compiled to static assets and embedded into the binary via `go:embed` (O-5: no SvelteKit — the UI stays Vite-only and embeddable). It is a pure client of the `/api/admin/*` surface; it holds no privileged logic, because every rule it reflects is server-enforced.

## Build and Embedding

- `web/` holds the Vite project; `vite build` emits `web/dist` with content-hashed asset filenames; the Go build embeds `web/dist`.
- The server serves hashed assets with `Cache-Control: public, max-age=31536000, immutable` and `index.html` with `no-cache` — the cache-busting contract lives in `09-deployment.md` (EC-15).
- The binary is the only deployment artifact; a UI change ships as a binary rebuild (N-1).

## SPA Conventions

- **State:** Svelte 5 runes (`$state`, `$derived`, `$effect`). Shared state lives in module-scope rune stores; no external state library.
- **API access:** one client module wraps `fetch`: it parses the response envelope of `04-api-layer.md`, converts `error.code` to typed failures, attaches `X-CSRF-Token` on every mutation, and redirects to login on `401`.
- **CSRF token:** delivered by login and `/api/admin/auth/csrf`, held in memory only — never `localStorage`, never a readable cookie (BR-AUTH-4). A hard refresh refetches it against the existing session.
- **Re-auth modal:** a `403` carrying the recent-auth detail triggers a password re-prompt, then retries the original request (BR-AUTH-5); destructive flows surface this before submission when the 4-hour window has lapsed.

## Screens

| Screen | Behavior |
|---|---|
| Schema builder | Collection and field CRUD over the operations of `03-dynamic-schema.md`. Destructive actions demand typed-slug confirmation client-side *and* server-side (BR-SCHEMA-7); type-change pickers offer only safe-matrix targets and explain rejections (EC-3). Rename dialogs state the API-path consequence (EC-4). |
| Content editor | Field widgets per type; Tiptap for `richText`, exchanging canonical JSONB with the API — the editor never produces HTML for storage. Field-level `validation_failed` details render inline at the offending widgets. |
| Revisions | Lists revisions per record, renders side-by-side compare of any two snapshots, restores through the drift-mapping flow with a preview of dropped/defaulted fields (EC-5, `03-dynamic-schema.md`). |
| Trash | Trash-scope listing, restore, and purge (purge is destructive → re-auth). Restore collisions surface the `409` field detail with a link to the colliding record (BR-LIFE-5). |
| Media library | Runs the presign → direct PUT → finalize flow of `04-api-layer.md` with client-side progress; abandoned uploads need no cleanup action — the orphan sweep owns them (EC-9). |
| Users & API keys | Role management within persona limits (P-2 cannot touch super admins); key creation shows the plaintext exactly once with a copy affordance (BR-AUTH-7). |
| Access rules | Per-collection rule editor for read/create/update/delete, predicates, and field-level visibility; renders the effective `Decision` per role as a preview matrix. |
| Setup (`/setup`) | Shown only on fresh systems (`cms_users` empty): accepts the logged single-use setup token and creates the first super admin (BR-AUTH-11). Returns 404 whenever `cms_users` is non-empty. |

## Editorial State Model

- The record header shows one of: **Draft** (never published), **Published**, **Published + pending draft** (newest `version_no` exceeds the published revision's — `07-data-model.md`).
- Publish/unpublish controls render only when the server would allow them (`publish` action probe via `Decision`); Contributors see their `ownerOnly` drafts and no publish control (BR-LIFE-3, BR-RBAC-2).
- **Conflict UX:** a `409` on save (BR-LIFE-7) opens a conflict dialog offering reload-and-reapply; there is no merge — real-time collaborative editing is out of scope (O-10).

## Security Posture

- Strict CSP: `default-src 'self'`; no inline scripts; Tiptap and all vendor code bundle at build time — the threat-model XSS mitigation depends on this header (`05-auth-security.md` §6).
- The SPA renders rich text from JSONB through Tiptap's schema-constrained renderer, never `innerHTML` of stored strings.
- Session cookie handling is entirely browser-managed (`HttpOnly`); the SPA never reads or writes it.
