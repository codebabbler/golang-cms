---
name: schema-ddl-safety
description: Use when creating or modifying anything in internal/schema (the runtime-DDL engine) — collection/field operations, DDL templates, slug validation, or type changes. Encodes the whitelist, identifier rules, advisory-lock discipline, and destructive-change gates.
---

# Schema DDL Safety

Distilled from `docs/architecture/03-dynamic-schema.md` and `docs/BUSINESS_RULES.md` §2. Those documents are authoritative; this skill is the working checklist.

## Hard Rules

1. **The whitelist is closed** (BR-SCHEMA-4). Exactly these operations exist: CreateCollection, DropCollection, RenameCollection, AddField, DropField, RenameField, ChangeFieldType, AddIndex, DropIndex, AddForeignKey, DropForeignKey. Never add a code path that executes DDL outside `schema.Engine.Apply`'s closed switch.
2. **Identifiers are never parameters and never raw** (BR-SCHEMA-3). Every identifier in a DDL template passes `query.Builder.QuoteIdent`. If you find yourself writing `fmt.Sprintf` with a table or column name that didn't come through `QuoteIdent`, stop.
3. **One transaction, one lock** (BR-SCHEMA-6). Every schema change: acquire `pg_advisory_xact_lock` on the global schema key → DDL → `cms_collections`/`cms_fields` metadata update → rebuild cache snapshot → commit → swap snapshot → release. The DDL and metadata must commit atomically; the cache must swap before the lock releases (BR-RUNTIME-7).
4. **Slug validation is double-layered** (BR-SCHEMA-2). `httpapi/admin.validateSlug` rejects at the boundary; the engine re-validates. Regex `^[a-z][a-z0-9_]{0,54}$`; blocklist covers Postgres reserved words, the seven system column names, and prefixes `cms_`, `c_`, `cj_`. Max 200 user fields per collection.
5. **The safe-conversion matrix is exhaustive** (BR-SCHEMA-5): `number` precision widening, and `{text, number, boolean, datetime, json, richText} → text` with `USING ::text`. `relation` and `media` never convert. Everything else: 422 with a drop-and-recreate remediation message.
6. **Destructive = gated** (BR-SCHEMA-7). DropField and DropCollection require `middleware.RequireRecentAuth` (4-hour window) AND typed confirmation of the exact slug in the request body. Never weaken either gate; never add a destructive operation without both.
7. **User columns are always nullable in DDL.** `required` lives in `content.Document.Set`, `unique` lives in partial unique indexes `WHERE deleted_at IS NULL` (`docs/architecture/07-data-model.md`).
8. **References use collection `id`, never slug.** Renames must stay a two-write transaction (slug + table rename); if a change forces a third write, the ID-reference rule is being violated somewhere (EC-4).

## Test Obligations

Every change here touches BR-SCHEMA rules — tests must trace: `TestBR_SCHEMA_6_DDLAndMetadataCommitAtomically`, `t.Run("BR-SCHEMA-5 rejects datetime→number", ...)`. Run `make trace` before considering the work done.

## Review Checklist

- [ ] New/changed DDL template quotes every identifier via `QuoteIdent`?
- [ ] Operation inside the closed `Apply` switch, advisory lock held?
- [ ] Metadata + DDL in the same transaction; cache swap before lock release?
- [ ] Destructive paths carry both gates?
- [ ] Conversion matrix untouched, or BUSINESS_RULES updated in the same change?
- [ ] `make trace` green?
