---
name: sqlc-workflow
description: Use when adding or changing a system table (cms_*) — migrations, sqlc queries, or regenerating store code. Encodes the migration → query → generate → test cycle and what must never go through sqlc.
---

# sqlc Workflow

Distilled from `docs/architecture/07-data-model.md` and `docs/architecture/10-project-structure.md`. Those documents are authoritative.

## Scope Boundary

sqlc covers **system tables only** (`cms_*`). Collection tables (`c_*`, `cj_*`) are dynamic and belong exclusively to `query.Builder` — never write a `store/queries` file touching them, and never let generated code reference them.

## The Cycle

1. **Migration:** add `internal/store/migrations/NNNN_description.sql` — next number, forward-only. No down migrations; recovery is restore-from-backup. Data transforms ship as SQL in the same file.
2. **Queries:** add or adjust `internal/store/queries/*.sql` for the new shape.
3. **Generate:** `make generate` (runs `sqlc generate`); commit the generated code — builds must need no toolchain beyond Go and Node.
4. **Test:** write the BR-traced test (integration tag if it needs PostgreSQL: `//go:build integration`).
5. **Verify:** `make test && make trace`.

## Table Rules (from the data model)

- All `id` columns are UUIDv7, generated application-side — no DB-side `gen_random_uuid()`.
- Tokens store hashed, never raw: `cms_sessions.token_hash`, `cms_api_keys.token_hash`, `cms_refresh_tokens.token_hash` (BR-AUTH-2, BR-AUTH-7).
- `cms_revisions` is append-only with the partial unique `(collection_id, record_id) WHERE published`; no UPDATE/DELETE queries against it except retention pruning (BR-LIFE-1).
- `cms_audit_log` (V2) exposes insert and select only — writing an UPDATE or DELETE query against it violates BR-AUDIT-3.
- References to collections use `cms_collections.id`, never slug (EC-4).
- Migrations run at startup under an advisory lock (`docs/architecture/09-deployment.md`); a migration must tolerate a concurrent waiting process — no advisory-lock acquisition inside migration SQL itself.

## Review Checklist

- [ ] Migration numbered, forward-only, self-contained?
- [ ] No collection-table reference anywhere in `store/`?
- [ ] Generated code committed in the same change?
- [ ] New columns respect the hashing and UUID rules above?
- [ ] BR-traced test added; `make trace` green?
