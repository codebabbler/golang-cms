# 06 — Admin UI

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

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
| Content editor | Field widgets per type; Tiptap for `richText`, exchanging canonical JSONB with the API — the editor never produces HTML for storage. Field-level `validation_failed` details render inline at the offending widgets. Lists request `?count=exact` where totals are displayed; cursor-aware next/prev navigation handles records beyond the 10,000-record offset ceiling (04-api-layer.md). |
| Revisions | Lists revisions per record, renders side-by-side compare of any two snapshots, restores through the drift-mapping flow with a preview of dropped/defaulted fields (EC-5, `03-dynamic-schema.md`). |
| Trash | Trash-scope listing, restore, and purge (purge is destructive → re-auth). Restore collisions surface the `409` field detail with a link to the colliding record (BR-LIFE-5). Restoring a published record displays a warning that it will become publicly visible immediately. |
| Media library | Runs the presign → direct PUT → finalize flow of `04-api-layer.md` with client-side progress; abandoned uploads need no cleanup action — the orphan sweep owns them (EC-9). |
| Users & API keys | Role management within persona limits (P-2 cannot touch super admins); key creation shows the plaintext exactly once with a copy affordance (BR-AUTH-7). Admins can issue one-time password-reset tokens to users (BR-AUTH-13); the token is displayed exactly once. API keys include a `passwordReset` capability toggle for fine-grained access control. |
| Access rules | Per-collection grant-matrix editor defining rules for read/create/update/delete/publish per role (minRole, minRoleOwn, endUsers, anonymous per action, per `12-access-rules.md`); renders the effective `Decision` per role as an interactive preview matrix. Field-level visibility is edited here too: `hideFrom`/`readOnlyFor` audience lists per field (`12-access-rules.md` §5). |
| Setup (`/setup`) | Shown only on fresh systems (`cms_users` empty): accepts the logged single-use setup token and creates the first super admin (BR-AUTH-11). Returns 404 whenever `cms_users` is non-empty. |
| Recovery (`/recover`) | Shown only when recovery mode is active (BR-AUTH-12); accepts the logged single-use token; 404 otherwise. |

## Editorial State Model

- The record header shows one of: **Draft** (never published), **Published**, **Published + pending draft** (newest `version_no` exceeds the published revision's — `07-data-model.md`).
- Publish/unpublish controls render only when the server would allow them (`publish` action probe via `Decision`); Contributors see their `ownerOnly` drafts and no publish control (BR-LIFE-3, BR-RBAC-2).
- **Conflict UX:** a `409` on save (BR-LIFE-7) opens a conflict dialog offering reload-and-reapply; there is no merge — real-time collaborative editing is out of scope (O-10).

## Security Posture

- Strict CSP: `default-src 'self'; img-src 'self' <media domain>; media-src 'self' <media domain>`; no inline scripts. The media domain is configured via the `R2_PUBLIC_BUCKET_URL` environment variable; the media domain must not share a registrable domain with the admin origin (`09-deployment.md`). Tiptap and all vendor code bundle at build time — the threat-model XSS mitigation depends on this header (`05-auth-security.md` §6).
- The SPA renders rich text from JSONB through Tiptap's schema-constrained renderer, never `innerHTML` of stored strings.
- Session cookie handling is entirely browser-managed (`HttpOnly`); the SPA never reads or writes it.
