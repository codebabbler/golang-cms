# golang-cms — Requirements

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This document defines what golang-cms delivers, for whom, and how delivery is verified. Functional requirements carry version tags; the system ships in three releases (V1 MVP, V2 Polish & Search, V3 Commerce & Video). Invariant-level rules live in `BUSINESS_RULES.md`; this document references them rather than restating them.

## 1. Product Definition

golang-cms is a headless, config-driven CMS delivered as a single Go binary serving a single tenant. Administrators define collections at runtime; the system provisions one real PostgreSQL table per collection through a whitelisted DDL engine. Content flows out through a REST API to two consumer classes — server-to-server integrations holding API keys and end-user client applications holding JWTs — while an embedded Svelte 5 admin UI (served from the binary via `go:embed`) manages schema, content, versioning, and access. PostgreSQL 16+ and S3-compatible object storage are the only runtime dependencies.

## 2. Personas

| ID | Name | Description |
|---|---|---|
| P-1 | Super Admin | Full system access. Manages users, roles, API keys, system configuration, and schema. |
| P-2 | Admin | Manages content and schema. Cannot rotate system keys or delete super admins. |
| P-3 | Editor | Creates and updates content. Cannot modify schema or access rules. |
| P-4 | Contributor | Can create content and edit own drafts. Cannot publish. |
| P-5 | Viewer | Read-only access to permitted collections and fields. |
| P-6 | API Consumer | External server-to-server actor authenticating via API Key. |
| P-7 | End User | Public client-app user authenticating via JWT. Interacts with public content or paywalled features. |

## 3. Functional Requirements

### Version 1 — MVP

- **F-1 (V1).** Super Admins and Admins create, rename, and delete collections at runtime; each collection materializes as a real Postgres table with the seven system columns.
- **F-2 (V1).** Collections support exactly eight field types: `text`, `richText`, `number`, `boolean`, `datetime`, `media`, `relation`, `json`, with the storage mapping defined in `docs/architecture/07-data-model.md`.
- **F-3 (V1).** Schema changes execute only whitelisted DDL operations; field type changes succeed only within the safe-conversion matrix; destructive changes demand recent re-authentication and typed confirmation. *(Resolves EC-2 via BR-SCHEMA-7.)*
- **F-4 (V1).** Record writes validate against the cached schema through `Document.Set`; unknown fields drop, role-read-only fields reject.
- **F-5 (V1).** Every record write produces a revision; the admin UI lists revisions, compares any two, and restores any revision as a new head.
- **F-6 (V1).** Records move through draft → published states; published records accept pending drafts without altering public output until republish; publish requires editor role or above. *(Resolves EC-7 via BR-LIFE-6.)*
- **F-7 (V1).** Deleting a record moves it to trash; trashed records vanish from all non-trash queries; restore returns them; purge removes them permanently subject to reference checks. *(Resolves EC-6 via BR-LIFE-5.)*
- **F-8 (V1).** Concurrent updates resolve by optimistic locking: a stale `version` yields `409 Conflict` and no write.
- **F-9 (V1).** Admins authenticate with email + Argon2id-hashed password, receive an HttpOnly session cookie plus CSRF token, and re-authenticate within 4 hours for destructive operations.
- **F-10 (V1).** Super Admins issue, scope, and revoke API keys; keys display once and authenticate server-to-server reads and writes per scope.
- **F-11 (V1).** The CMS acts as the user store for client applications: end users register and log in, receive a 15-minute RS256 JWT plus rotating refresh token, and lose the whole token family on refresh-token reuse. *(Resolves EC-8 via BR-AUTH-9.)*
- **F-12 (V1).** Per-collection access rules (read/create/update/delete/publish) with row predicates (e.g., `ownerOnly`) and field-level visibility (`hideFrom`, `readOnlyFor` audience lists) gate every request; missing rules deny for the governed classes.
- **F-13 (V1).** Media uploads flow directly to object storage via presigned PUT URLs with size caps; the binary finalizes media records and serves delivery URLs.
- **F-14 (V1).** The embedded admin UI covers schema building, content editing with Tiptap rich text (stored as JSONB), revision management, trash, user/key management, and access-rule editing.
- **F-15 (V1).** List endpoints paginate with capped limit/offset and a stable sort; excessive offsets reject. *(Resolves EC-11 via BR-API-1.)* Admin lists additionally support keyset cursor pagination from V1.
- **F-16 (V1).** Every mutation emits an audit event through the audit interface; the V1 sink is structured logging. V1 audit durability equals shipped stdout logs; queryable persistence arrives in V2 (accepted limitation).
- **F-17 (V1).** The binary runs embedded system-table migrations automatically at startup under an advisory lock.
- **F-32 (V1).** Account recovery and password reset: env-gated super-admin recovery mode (BR-AUTH-12); admin-issued one-time reset tokens; end-user reset via API-key-gated request plus public confirm, revoking all refresh-token families on success (BR-AUTH-13). The binary never sends email.

### Version 2 — Polish & Search

- **F-18 (V2).** Postgres FTS indexes selected fields per collection with per-field weights, including plaintext extracted from Tiptap JSONB; search endpoints return ranked results.
- **F-19 (V2).** Public reads accept `?format=html` to render `richText` fields to HTML server-side; JSONB remains canonical.
- **F-20 (V2).** Records carry SEO metadata (title, description, canonical, open-graph fields) exposed distinctly in API responses.
- **F-21 (V2).** Admins manage 301/302 redirects; a public lookup endpoint resolves paths for the consuming site.
- **F-22 (V2).** Webhooks fire on lifecycle events (create, update, publish, trash) with HMAC-signed payloads and bounded in-process retry.
- **F-23 (V2).** Relations extend to many-to-many with join tables and dedicated admin UI.
- **F-24 (V2).** Audit events persist to `cms_audit_log` with an admin UI for filtering by actor, action, entity, and time.
- **F-25 (V2).** Schema export/import and content export are first-class; content import is best-effort with documented limitations (ID collisions, relation remapping, media references).
- **F-26 (V2).** Records publish automatically at `publish_at`; missed schedules publish at next startup. *(Resolves EC-13 via BR-LIFE-9.)*
- **F-27 (V2).** The public API exposes the V1 keyset-cursor mechanism alongside capped offset pagination.
- **F-33 (V2).** GDPR-class erasure: end-user hard delete with token-family revocation, `created_by` anonymization to a tombstone UUID, and a revision-redaction capability in the retention job, with documented limitations.

### Version 3 — Commerce & Video

- **F-28 (V3).** End users hold server-side cart state attached to their JWT identity.
- **F-29 (V3).** Stripe integration covers checkout sessions and webhook-driven order state; orders persist as system records.
- **F-30 (V3).** Video content gates behind paywalls: entitled end users receive short-lived signed playback tokens; transcoding and packaging are delegated to the video provider.
- **F-31 (V3).** Custom Tiptap blocks embed product and video references in rich text; the public API resolves them to structured data.

## 4. Non-Functional Requirements

- **N-1.** The deliverable is one statically linked binary embedding all UI assets and migrations; deployment requires the binary plus environment variables, nothing else.
- **N-2.** The binary targets linux/amd64 and linux/arm64.
- **N-3.** Published single-record fetch answers in ≤ 50 ms at p95 with 100,000 records per collection on a 2 vCPU / 4 GB reference host.
- **N-4.** List queries (limit 25, indexed sort) answer in ≤ 120 ms at p95 at the same scale.
- **N-5.** Startup completes in ≤ 2 seconds when migrations are current; a schema change becomes visible to requests within 1 second of commit.
- **N-6.** Graceful shutdown drops zero accepted requests within the 15-second drain window.
- **N-7.** All security invariants in `BUSINESS_RULES.md` §4–§5 (Argon2id, hashed tokens, CSRF, RS256, default-deny RBAC, rate limits) hold in every build; the threat model of the auth specification is fully mitigated.
- **N-8.** Every request logs one structured `slog` line with request ID, principal, route, status, and latency; log level follows `CMS_LOG_LEVEL`.
- **N-9.** Revision history and trash respect the `CMS_REVISION_LIMIT` and `CMS_TRASH_RETENTION_DAYS` bounds; the database schema remains restorable by plain `pg_dump`/`pg_restore` with no non-bundled extensions.
- **N-10.** CI enforces Rule-to-Code traceability: every non-structural BR identifier maps to at least one named test.
- **N-11.** The system fails closed: on schema-cache or access-rule load failure, affected requests return errors rather than serving stale or permissive results.
- **N-12.** RPO ≤ 5 minutes and RTO ≤ 1 hour via WAL-based point-in-time recovery; scheduled `pg_dump` remains the portable second copy; a restore drill precedes V1 release.
- **N-13.** The service is single-instance: upgrades cost drain-plus-startup downtime and the availability class is ~99.5%; high availability is out of scope.

## 5. Out of Scope

- **O-1.** Multi-tenancy — the binary is single-tenant.
- **O-2.** SSO / OIDC, 2FA, Passkeys.
- **O-3.** Cross-site cookie sharing.
- **O-4.** Non-PostgreSQL database backends.
- **O-5.** SvelteKit — the UI remains Vite-only and embeddable.
- **O-6.** i18n / localization — adding it later is a major-version undertaking; documents must not design for it speculatively.
- **O-7.** In-binary image transformation — delegated to Cloudflare Image Resizing.
- **O-8.** In-binary video transcoding — delegated; the binary signs playback tokens only.
- **O-9.** External runtime dependencies — no Redis, message queues, or external caches.
- **O-10.** Real-time collaborative editing — optimistic locking rejects conflicts; no CRDT.
- **O-11.** Plugin runtime — webhooks are the only extensibility surface.
- **O-12.** GraphQL — the API is REST-only.

## 6. Acceptance Criteria

### Version 1

- **UAC-1.1.** A Super Admin logs in, creates collection `posts` containing one field of every one of the eight types, and the system provisions table `c_posts` with those columns plus the seven system columns; the admin API returns the schema definition, and `psql \d c_posts` confirms the physical columns.
- **UAC-1.2.** A Contributor creates a draft post and cannot publish it (403); an Editor publishes it; an API Consumer with a read-scoped key fetches it; the Contributor's subsequent edit to another user's record is denied by `ownerOnly`.
- **UAC-1.3.** An Editor edits a published record producing a pending draft — the public API continues serving the published content unchanged — then republishes; the revision list shows both versions, a compare renders the differences, and restoring the older revision creates a new head without rewriting history.
- **UAC-1.4.** Deleting a published record removes it from every list, read, search, and relation resolution; restoring returns it intact; a concurrent update presenting a stale `version` receives `409 Conflict` and changes nothing.
- **UAC-1.5.** A client requests a presigned URL, uploads a file directly to storage, finalizes the media record, and attaches it to a `media` field; separately, an End User logs in, refreshes tokens once, then replays the old refresh token and receives `401` with the whole token family revoked.
- **UAC-1.6.** An API consumer whose key carries `passwordReset` requests a reset for an end user and receives a token; confirmation sets the new password and revokes every refresh-token family; separately, a recovery-mode start (`CMS_RECOVERY_EMAIL`) logs a single-use token that resets a locked-out super admin and dies on use.

### Version 2

- **UAC-2.1.** Searching a collection returns ranked matches from both a `text` title and plaintext inside a Tiptap `richText` body; adding a field to the search config triggers reindexing and the field becomes searchable.
- **UAC-2.2.** Exporting the schema from instance A and importing it into a fresh instance B yields identical collection definitions (verified by diffing both schema exports); a publish event delivers an HMAC-signed webhook to a registered endpoint within 30 seconds.
- **UAC-2.3.** A record with `publish_at` set 2 minutes ahead publishes automatically; a record whose `publish_at` elapsed while the binary was stopped publishes during the next startup.
- **UAC-2.4.** A schema change, a publish, and an API-key revocation each appear in the audit UI with actor, action, entity, and timestamp; a configured 301 redirect resolves through the lookup endpoint with its target and status code.

### Version 3

- **UAC-3.1.** An End User adds two products to a cart, completes a Stripe test-mode checkout, and an order record persists with line items matching the cart.
- **UAC-3.2.** An anonymous request for a paywalled video's playback token receives `403`; an entitled End User receives a signed playback token that plays, and the same token fails after expiry.
- **UAC-3.3.** An Editor embeds a product block in a post's rich text; the public API response resolves the block to the product's structured data, and deleting that product renders the block reference `null` without breaking the post.
