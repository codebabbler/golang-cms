# golang-cms

Headless, config-driven CMS: one Go binary, one tenant. Admins define collections at runtime; the schema engine provisions real Postgres tables. PostgreSQL 16+ and S3-compatible storage are the only runtime dependencies. Status: documentation complete (`docs/`), implementation not started.

## Authority Chain

`docs/BUSINESS_RULES.md` > `docs/architecture/*` > `.claude/skills/*` > code comments. On conflict, the higher source wins and the lower one gets fixed.

## Documentation Map

| Doc | Contents |
|---|---|
| `docs/BUSINESS_RULES.md` | All invariants (BR-IDs) with enforcement points; tests trace to these |
| `docs/REQUIREMENTS.md` | PRD: personas, F/N/O requirements, acceptance criteria (UAC) |
| `docs/architecture/00-README.md` | Reading order and identifier conventions |
| `docs/architecture/01-system-overview.md` | Architecture, request flows, Edge-Case Register (EC-1…16) |
| `docs/architecture/02-core-interfaces.md` | QueryBuilder, Decision evaluator, Document.Set, service contracts |
| `docs/architecture/03-dynamic-schema.md` | Runtime-DDL engine: whitelist, conversion matrix, drift mapping |
| `docs/architecture/04-api-layer.md` | Routes, middleware order, envelope, pagination, uploads |
| `docs/architecture/05-auth-security.md` | Sessions, API keys, JWTs, RBAC, bootstrap, threat model |
| `docs/architecture/06-admin-ui.md` | Svelte 5 SPA conventions and screens |
| `docs/architecture/07-data-model.md` | System tables, collection anatomy, live-table/revisions contract |
| `docs/architecture/08-observability.md` | slog schema, audit stream, job telemetry |
| `docs/architecture/09-deployment.md` | Build, startup/shutdown, proxy contract, cache busting |
| `docs/architecture/10-project-structure.md` | Package layout, import rules, Makefile |
| `docs/architecture/11-roadmap.md` | V1/V2/V3 scopes and delivery gates |
| `docs/architecture/12-access-rules.md` | Grant-matrix schema, evaluation algorithm, audiences, API-key scopes |

## Skills (invoke before touching the subsystem)

| Skill | Fires when |
|---|---|
| `schema-ddl-safety` | `internal/schema` — DDL engine, slugs, type changes |
| `query-builder-invariants` | `internal/query` or any collection list/read path |
| `content-lifecycle-invariants` | `internal/lifecycle` — saves, publish, trash, restore |
| `auth-security-conventions` | `internal/auth` — tokens, hashing, credentials, bootstrap |
| `api-conventions` | `internal/httpapi` — handlers, routes, middleware |
| `sqlc-workflow` | `cms_*` system tables, migrations, `internal/store` |
| `svelte-admin-conventions` | `web/` — admin UI |
| `br-traceability-testing` | writing tests or changing BUSINESS_RULES.md |
| `run-and-verify` | running, smoke-testing, or verifying the app locally |

## Hard Rules (always on)

1. SQL for collection tables exists only inside `internal/query` — BR-SCHEMA-3.
2. Every content write goes through `content.Document.Set` — BR-RBAC-5.
3. DDL runs only through `schema.Engine`'s closed whitelist — BR-SCHEMA-4.
4. Error responses only via `httpapi.WriteError` with the closed code registry — BR-API-3.
5. No runtime dependencies beyond PostgreSQL 16+ and S3-compatible storage — BR-RUNTIME-2.
6. Never log tokens, cookie values, presigned URLs, or JWT bodies — sole exceptions: the single-use setup/recovery tokens (BR-AUTH-11/12), logged once at warn with a 30-minute TTL, emitted regardless of log level.

## Workflow

- Commands: `make build` (vite→go), `make test` (unit+integration, disposable PG16), `make trace` (BR coverage gate), `make generate` (sqlc), `make dev`.
- Git: the user commits; never commit or stage unless explicitly asked.
- Flow: brainstorm → spec (`docs/superpowers/specs/`) → plan (`docs/superpowers/plans/`) → subagent-driven execution for code.
- Done = smoke-relevant checks pass AND `make test && make trace` green — not "it compiles."
