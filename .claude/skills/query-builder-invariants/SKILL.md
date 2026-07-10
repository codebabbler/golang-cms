---
name: query-builder-invariants
description: Use when touching internal/query (the collection query builder) or any handler that lists/reads collection records — filtering, sorting, pagination, scopes, or predicates. Encodes the six caller-proof invariants that make queries safe by construction.
---

# Query Builder Invariants

Distilled from `docs/architecture/02-core-interfaces.md` (query.Builder contract), `docs/architecture/03-dynamic-schema.md`, and `docs/architecture/04-api-layer.md`. Those documents are authoritative.

## The Six Invariants (callers cannot disable any)

1. **Quoting:** every identifier passes `QuoteIdent`; identifiers are never SQL parameters, values are never interpolated (BR-SCHEMA-3).
2. **Trash filter:** `deleted_at IS NULL` appends to every query except `ScopeTrash` (BR-LIFE-4). A missed trash filter is a data leak of "deleted" content — this is why the predicate lives in the builder, not in handlers.
3. **Public scope:** `ScopePublic` appends `status = 'published'` (BR-API-2).
4. **Decision predicate:** `WithDecision` compiles the row predicate (e.g., `ownerOnly` → `created_by = principal.ID`) into the query (BR-RBAC-6). The builder constructor requires a `Decision` for collection scopes — keep the bypass unrepresentable.
5. **Pagination clamps:** limit defaults 25, clamps to 100; offset > 10,000 rejects; the `id` tiebreaker sort always appends (BR-API-1, EC-11).
6. **Schema-checked fields:** `Where`/`Sort` reject fields absent from the schema snapshot or hidden by `Decision.FieldRules`. Operators are the closed set `eq neq lt lte gt gte in contains`; `contains` maps to `ILIKE` with escaped wildcards on `text` fields only.

## Working Rules

- `squirrel` is an implementation detail. It imports **only** inside `internal/query`; a second import site anywhere breaks the replaceability guarantee (`docs/architecture/10-project-structure.md`).
- Handlers never build SQL. If a handler needs a new query shape, extend the builder contract — don't open an escape hatch.
- Collection tables never appear in `internal/store/queries` (sqlc is for system tables only).
- Relation expansion loads targets through `ScopePublic` on public routes; trashed/draft/hidden targets serialize as `null`, never as errors (BR-LIFE-6, EC-7).

## Test Obligations

Builder changes trace to BR-SCHEMA-3, BR-LIFE-4, BR-API-1, BR-API-2, BR-RBAC-6. Highest-value tests: adversarial identifiers (`posts"; DROP TABLE`), scope-escape attempts, clamp boundaries (limit 101, offset 10_001), and hidden-field filter attempts. Run `make trace`.

## Review Checklist

- [ ] No new interpolation path outside `QuoteIdent`?
- [ ] All six invariants still apply to every `Build()` output, including new query shapes?
- [ ] No `squirrel` import outside `internal/query`?
- [ ] Boundary tests updated for any changed clamp or operator?
