# golang-cms — Architecture Documentation

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This directory holds the architecture documentation for golang-cms: a headless, config-driven CMS delivered as a single Go binary for a single tenant. The documents below, together with the two root contracts, form the complete design record. Implementation must conform to them; where a document and `../BUSINESS_RULES.md` disagree, the business rule wins.

## Document Map

| Document | Contents |
|---|---|
| `../BUSINESS_RULES.md` | The invariants manual. Every non-negotiable rule with its enforcement point (BR identifiers). |
| `../REQUIREMENTS.md` | The PRD. Personas, functional/non-functional requirements, out of scope, acceptance criteria (F/N/O/UAC identifiers). |
| `01-system-overview.md` | The single-binary architecture, request flows, and the Edge-Case Register with its citation convention. |
| `02-core-interfaces.md` | The internal seams: `QueryBuilder`, the access `Decision` evaluator, `Document.Set`, the audit recorder. |
| `03-dynamic-schema.md` | The runtime-DDL engine: slugs, whitelisted operations, the safe-conversion matrix, transactional DDL. |
| `04-api-layer.md` | HTTP surface: routing, middleware order, pagination, the error envelope, presigned-upload endpoints. |
| `05-auth-security.md` | The three principals (sessions, API keys, JWTs), RBAC, and the threat model. |
| `12-access-rules.md` | The access-control grant matrix: schema, evaluation algorithm, audiences, API-key scopes. |
| `06-admin-ui.md` | The embedded Svelte 5 SPA: build, embedding, Tiptap, versioning/trash UI, CSRF handling. |
| `07-data-model.md` | Physical storage: system tables, collection tables, the live-table/revisions contract, retention. |
| `08-observability.md` | Structured logging conventions, request correlation, audit event stream. |
| `09-deployment.md` | Build, configuration, startup/shutdown, backup, and the reverse-proxy contract. |
| `10-project-structure.md` | Go module layout and package responsibilities. |
| `11-roadmap.md` | The three release scopes and their sequencing rationale. |
| `../api/openapi.yaml` | OpenAPI 3.1 specification for the `/api/v1/` REST API. |

## Reading Order

- **New engineer (full pass):** `01` → `07` → `03` → `02` → `04` → `05` → `12` → `06` → `08` → `09` → `10` → `11`, with `../BUSINESS_RULES.md` open throughout.
- **Backend work:** `01` → `07` → `03` → `02` → `04` → `05` → `12`.
- **Admin UI work:** `01` → `04` → `05` → `06`.
- **Operations:** `01` → `09` → `08`.

## Identifier Conventions

- **BR-[DOMAIN]-[#]** — business rule (`../BUSINESS_RULES.md`). Tests trace to these names.
- **F-[#] / N-[#] / O-[#]** — functional, non-functional, out-of-scope requirement (`../REQUIREMENTS.md`).
- **UAC-[v].[#]** — acceptance criterion per version (`../REQUIREMENTS.md` §6).
- **EC-[#]** — edge case from the Edge-Case Register (`01-system-overview.md`). Every architecture document states which EC items it resolves and how, using the form *(Resolves EC-n.)*
