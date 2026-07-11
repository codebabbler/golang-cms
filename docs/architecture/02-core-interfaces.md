# 02 — Core Interfaces

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This document specifies the internal seams of the binary: the contracts each package exposes, the invariants each contract owns, and the dependency rules between them. Handlers depend on these interfaces only — never on implementations. Signatures appear at contract level; implementation belongs to the delivery cycle.

## Dependency Direction

```text
httpapi ──► access.Evaluator ──► schema.Cache
   │              │
   ├──► content.Document ──► schema.Cache
   ├──► lifecycle.Service ──► query.Builder ──► schema.Cache
   ├──► media.Service
   ├──► auth.{SessionService, APIKeyService, JWTService, ResetService}
   └──► audit.Recorder
```

Nothing depends on `httpapi`. `query.Builder` is the only package that renders SQL for collection tables; `sqlc`-generated code covers system tables only.

## Principal

Every authenticated request resolves to one `Principal` threaded through `context.Context`:

```text
Principal { Kind: admin | api_key | end_user | anonymous
            ID:   UUID (zero for anonymous)
            Role: super_admin | admin | editor | contributor | viewer (admin kind only)
            Scopes: []CollectionScope (api_key kind only) }
```

`middleware.Auth` produces it; nothing downstream re-authenticates (BR-AUTH-8: permissions resolve server-side per request, from the Principal, never from token claims).

## access.Evaluator

```text
Decide(p Principal, collection Collection, action Action, record *RecordMeta) Decision
Decision { Allowed bool
           Predicate  Predicate      // row filter, e.g. ownerOnly → created_by = p.ID
           FieldRules FieldRuleSet } // hideFrom / readOnlyFor resolution
```

Actions: `read`, `create`, `update`, `delete`, `publish`. Guarantees: a missing rule denies (BR-RBAC-3); `publish` requires editor or above (BR-LIFE-3); `Decision.Predicate` is opaque to handlers and compiles only inside `query.Builder.WithDecision` (BR-RBAC-6). Handlers must call `Decide` before touching `query.Builder`; the builder constructor requires a `Decision` for collection scopes, making the bypass unrepresentable (BR-RBAC-2). Grant semantics, audiences, and scope shapes are normative in 12-access-rules.md (BR-RBAC-7).

## query.Builder

The only SQL surface for collection tables. V1 backs it with `squirrel`; the backing library is replaceable behind this contract without touching callers.

```text
ForCollection(snap SchemaSnapshot, col Collection, scope Scope) Builder
   Scope: ScopeAdmin | ScopePublic | ScopeTrash
WithDecision(d Decision) Builder
Where(field Slug, op Op, value any) Builder     // op ∈ eq neq lt lte gt gte in contains
Sort(field Slug, dir Dir) Builder
Paginate(p Page) Builder
Build() (sql string, args []any, err error)
```

Invariants the builder owns — callers cannot disable any of them:

1. Every identifier passes `QuoteIdent`; identifiers are never parameters, values are never interpolated (BR-SCHEMA-3).
2. `deleted_at IS NULL` appends to every query except `ScopeTrash` (BR-LIFE-4).
3. `ScopePublic` appends `status = 'published'` (BR-API-2).
4. `WithDecision` appends the row predicate (BR-RBAC-6).
5. `Paginate` clamps limit to 100, rejects offset > 10,000, and always appends the `id` tiebreaker sort (BR-API-1); `Page` carries either an `offset` or an opaque cursor of sort key + `id` — supplying both is a `422` error.
6. `Where`/`Sort` reject fields absent from the schema snapshot or hidden by `Decision.FieldRules`.
7. `ScopePublic` accepts filter/sort only on indexed or unique fields (BR-API-4); cursor pagination preserves the `id` tiebreaker and limit clamps.

## content.Document

```text
Set(snap SchemaSnapshot, col Collection, rules FieldRuleSet, input map[string]any) (Document, []FieldError)
```

The only write path into collection tables (BR-RBAC-5). Guarantees: unknown fields drop silently; `readOnlyFor` fields reject with field-level errors; type validation follows the field-type reference (numbers, datetimes, Tiptap JSONB shape for `richText`, finalized-media references for `media` — BR-MEDIA-3); `required` enforcement happens here, not in DDL (`03-dynamic-schema.md`). Any single field value caps at 1 MiB; larger values reject with a field-level error.

## lifecycle.Service

```text
Save(doc Document, expectedVersion int64) (Record, error)   // 409 on version mismatch
Publish(recordID UUID, revisionNo int64) error               // editor+
Unpublish(recordID UUID) error
Trash(recordID UUID) error
Restore(recordID UUID) error                                  // 409 on unique collision
RestoreRevision(recordID UUID, versionNo int64) (Record, error)
Purge(recordID UUID) error                                    // FK RESTRICT enforced
Revisions(recordID UUID, page Page) ([]Revision, error)
```

Guarantees: `Save` writes live row + revision in one transaction (BR-LIFE-1); `Publish` copies revision data into the live row and moves the `published` flag atomically (BR-LIFE-2); `RestoreRevision` applies the drift-mapping rules of `03-dynamic-schema.md`; every method emits through `audit.Recorder` (BR-AUDIT-1).

## media.Service

```text
Presign(p Principal, req UploadRequest) (PresignedUpload, error)  // ≤15 min, size-capped
Finalize(p Principal, mediaID UUID) (Media, error)                 // verifies object existence
```

Guarantees: no method accepts file bytes (BR-MEDIA-1); `Finalize` flips `pending → finalized` only after a storage HEAD confirms the object within the declared size (BR-MEDIA-2).

## auth Services

| Service | Contract | Guarantees |
|---|---|---|
| `auth.SessionService` | `Issue(user) (cookie, csrfToken)` · `Verify(cookieValue) (Session, error)` · `Destroy(cookieValue)` | Hashed storage both ways (BR-AUTH-2); cookie attributes per BR-AUTH-1. |
| `auth.APIKeyService` | `Create(name, scopes) (plaintextOnce, Key)` · `Verify(bearer) (Key, error)` · `Revoke(id)` | sha256 at rest; revoked rows persist (BR-AUTH-7). |
| `auth.JWTService` | `Issue(endUser) (jwt, refresh)` · `Verify(jwt) (Claims, error)` · `Refresh(refreshToken) (jwt, refresh, error)` | RS256, 15-min TTL, identity-only claims (BR-AUTH-8); rotation + family revocation on reuse (BR-AUTH-9); kid header (BR-AUTH-10). |
| `auth.ResetService` | `Request(kind, userID) (plaintextOnce, ResetToken)` · `Confirm(token, newPassword) error` | Hashed at rest, 30-min TTL, single-use; Confirm revokes all refresh-token families (BR-AUTH-13). |

## audit.Recorder

```text
Emit(e Event)   Event { Actor Principal; Action string; Entity EntityRef; Detail map[string]any; At time.Time }
Sink interface { Write(Event) error }
```

Call sites are wired in V1 across every mutation path (BR-AUDIT-1); V1 configures the `slog` sink, V2 adds the `cms_audit_log` sink behind the same interface without touching call sites (BR-AUDIT-2).

## Stability Rules

- Adding a method to these interfaces is a minor change; changing a signature or weakening a guarantee requires a business-rule review because tests trace to BR identifiers (N-10).
- `squirrel` (inside `query.Builder`) and the storage SDK (inside `media.Service`) are the two vendored decisions explicitly marked replaceable; no other package may import them.
