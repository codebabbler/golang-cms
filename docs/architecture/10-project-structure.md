# 10 — Project Structure

**Version:** 1.0 · **Last Updated:** 2026-07-08 · **Owner:** Miraj Aryal

One Go module, everything importable under `internal/`, one command. The layout mirrors the interface seams of `02-core-interfaces.md` — one package per contract — so the dependency rules there are enforceable by import review.

## Layout

```text
golang-cms/
├── cmd/
│   └── cms/
│       └── main.go            # flag-free: env config only; calls app.Run
├── internal/
│   ├── app/                   # wiring, startup order (BR-RUNTIME-3), shutdown drain
│   ├── httpapi/               # chi router, handlers, middleware, envelope (WriteError)
│   ├── access/                # Evaluator, Decision, predicates, role constants
│   ├── schema/                # Engine (runtime DDL), Cache (atomic snapshot)
│   ├── query/                 # Builder — the only collection-SQL surface
│   ├── content/               # Document.Set validation
│   ├── lifecycle/             # Save/Publish/Trash/Restore/Purge, revisions
│   ├── media/                 # Presign/Finalize, storage SDK confined here
│   ├── auth/                  # SessionService, APIKeyService, JWTService, Keys, Password
│   ├── audit/                 # Recorder, Event, sinks (slog V1, DB V2)
│   ├── jobs/                  # Scheduler, Retention, Publisher tickers
│   └── store/                 # sqlc-generated system-table queries
│       ├── migrations/        # embedded *.sql, ordered, forward-only
│       └── queries/           # sqlc source .sql files
├── web/                       # Svelte 5 + Vite SPA (06-admin-ui.md)
│   ├── src/
│   └── dist/                  # build output, go:embed target (gitignored)
├── docs/                      # this documentation set
├── sqlc.yaml
├── Makefile                   # build (vite → go), test, trace, generate
└── go.mod
```

## Package Rules

| Rule | Rationale |
|---|---|
| Nothing imports `httpapi`. | Handlers are the composition root's leaves; inverting this couples domain logic to HTTP (`02-core-interfaces.md` dependency direction). |
| `squirrel` imports only in `query`. | The builder is the replaceable seam; a second import site breaks the swap guarantee. |
| The storage SDK imports only in `media`. | Same replaceability rule. |
| `sqlc` output imports only via `store`; collection tables never appear in `store/queries`. | System tables are static and type-safe; collection tables are dynamic and belong to `query.Builder` exclusively. |
| DDL strings exist only in `schema` (engine templates) and `store/migrations`. | Two auditable DDL surfaces — one dynamic and whitelisted (BR-SCHEMA-4), one static and migrated. |
| No imaging or media-processing libraries in `go.mod`. | BR-MEDIA-4 [structural]; CI dependency review enforces BR-RUNTIME-2. |

## Migrations

Forward-only, numbered `NNNN_description.sql`, embedded via `go:embed`, executed at startup under an advisory lock (`09-deployment.md`). No down migrations: recovery is restore-from-backup, matching the single-tenant operational model. A migration that must transform data ships as SQL in the same file — the binary never runs ad-hoc data scripts.

## Code Generation

- `make generate` runs `sqlc generate`; generated code commits to the repository so builds need no toolchain beyond Go and Node.
- The `sqlc` workflow for a system-table change: add migration → add/adjust `store/queries/*.sql` → `make generate` → write the BR-traced test → `make test`.

## Testing Layout (N-10)

- Unit tests live beside their package (`internal/schema/engine_test.go`).
- Integration tests requiring PostgreSQL live in the package with an `//go:build integration` tag; `make test` runs both against a disposable database.
- Test names trace to business rules per `../BUSINESS_RULES.md` § Rule-to-Code Traceability: `TestBR_SCHEMA_6_DDLAndMetadataCommitAtomically`, or `t.Run("BR-LIFE-5 ...")`. The `make trace` target greps the manual's identifiers against `_test.go` files and fails on uncovered non-structural rules.

## Makefile Targets

| Target | Does |
|---|---|
| `build` | `vite build` → `go build` with embeds; fails on missing `web/dist` (`09-deployment.md`). |
| `test` | Unit + integration against a disposable PostgreSQL 16. |
| `trace` | BR coverage check (N-10). |
| `generate` | `sqlc generate`. |
| `dev` | Vite dev server proxying `/api` to a locally running binary — the only place UI and API run unembedded. |
