# 12 — Access Rules

**Version:** 1.0 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

This document is the grant-matrix specification for `cms_collections.access_rules` (BR-RBAC-2) — the closed vocabulary that BR-RBAC-7 requires every access-rule object to conform to. `access.Evaluator.Decide` (`02-core-interfaces.md`) implements the algorithm in §3 against the schema in §1; `httpapi/admin` enforces the validation in §4 at write time.

## 1. Grant-Matrix Schema (BR-RBAC-2, BR-RBAC-3, BR-LIFE-3)

`cms_collections.access_rules` is a JSONB object keyed by action: `read`, `create`, `update`, `delete`, `publish`. Each value is a **grant object** with only these keys, all optional:

| Key | Type | Meaning |
|---|---|---|
| `minRole` | role name | Admin principals with role ≥ this get the action **unrestricted** |
| `minRoleOwn` | role name | Admin principals with role ≥ this (and < `minRole` if both set) get the action **with `ownerOnly`** (`created_by = principal.ID`). Must be ≤ `minRole` in the lattice |
| `endUsers` | `"own"` \| `"all"` | End-user principals: `all` = unrestricted (within ScopePublic on reads), `own` = `ownerOnly`. Omitted = deny |
| `anonymous` | bool | Valid on `read` only. `true` = anonymous read under ScopePublic |

**Role lattice:** `viewer < contributor < editor < admin < super_admin`.

**Default deny (BR-RBAC-3):** an omitted action key denies that action for all governed classes — `editor`, `contributor`, `viewer`, `end_user`, `anonymous`. `super_admin` and `admin` are not governed by the matrix; they hold an implicit full grant (§3 step 1).

**Publish floor (BR-LIFE-3):** `publish` additionally floors at `editor` regardless of rules. A `publish` grant object can restrict further (e.g., pair `minRoleOwn` with a higher `minRole`) but can never lower access below `editor` — no rule shape grants `publish` to `contributor` or `viewer`.

## 2. Worked Examples (Normative)

These three examples fix the schema's intended usage; an implementation that cannot express them is non-conformant.

```jsonc
// Public blog: world-readable, editors run it
{ "read":    { "minRole": "viewer", "anonymous": true, "endUsers": "all" },
  "create":  { "minRole": "editor" },
  "update":  { "minRole": "editor" },
  "delete":  { "minRole": "editor" },
  "publish": { "minRole": "editor" } }
```

```jsonc
// Comments: end users own their rows; contributors moderate their own; editors moderate all
{ "read":    { "minRole": "viewer", "anonymous": true, "endUsers": "all" },
  "create":  { "endUsers": "all", "minRole": "contributor" },
  "update":  { "endUsers": "own", "minRoleOwn": "contributor", "minRole": "editor" },
  "delete":  { "minRole": "editor", "endUsers": "own" } }
```

```jsonc
// Ingestion target: no browser access at all; a write-scoped API key feeds it
{ "read":    { "minRole": "viewer" },
  "create":  {},              // admins implicit; key scope grants the rest
  "update":  {},
  "delete":  { "minRole": "admin" } }
```

## 3. Evaluation Algorithm (BR-RBAC-3, BR-RBAC-6)

`access.Evaluator.Decide` resolves a `(principal, collection, action)` triple in this order:

1. `super_admin`/`admin` → `Decision{Allowed: true}` for all content actions. Field rules still apply below `super_admin`.
2. Admin roles: `role ≥ minRole` → allowed unrestricted; else `role ≥ minRoleOwn` → allowed with `Predicate: ownerOnly`; else deny.
3. `end_user`: per `endUsers` (`own` ⇒ `ownerOnly`); reads additionally scoped by ScopePublic (BR-API-2).
4. `anonymous`: `read` + `anonymous: true` only, always ScopePublic.
5. `api_key`: allowed iff the key's scopes contain `(collection, action)`; reads published-only unless `draftAccess` (§6). Revoked keys resolve to `anonymous`.
6. Field rules resolve per §5 and ride along in `Decision.FieldRules`.

Each principal class is resolved by exactly one of steps 1–5; the numbering is evaluation order, not a priority among rules that could simultaneously apply within one class. `Decision.Predicate` compiles into the query via `query.Builder.WithDecision` (BR-RBAC-6).

For `action = publish`, step 2 is bounded below by the floor in §1: no `minRole`/`minRoleOwn` combination admits a role below `editor`, and `end_user`, `anonymous`, and `api_key` principals never receive `publish` (BR-LIFE-3).

## 4. Validation and Fail-Closed Reads (BR-RBAC-7)

**Write-time** (`httpapi/admin` + evaluator re-check): a rule write is rejected with `422 validation_failed` naming the offending path when the payload contains any of:

- an unknown action key (outside `read`, `create`, `update`, `delete`, `publish`),
- an unknown grant key (outside `minRole`, `minRoleOwn`, `endUsers`, `anonymous`),
- an unknown role name (outside the five-role lattice, for `minRole` or `minRoleOwn`),
- `minRoleOwn` set higher than `minRole` when both are set (`minRoleOwn > minRole`),
- `anonymous` present on any action other than `read`, or
- a non-boolean `anonymous` value or a non-enum `endUsers` value.

**Fail-closed at read (N-11):** rules that fail to parse at evaluation time deny. A malformed `access_rules` value never falls back to permissive behavior — `access.Evaluator.Decide` returns `Decision{Allowed: false}` for every request against that collection until an admin corrects the rule.

## 5. Field-Rule Audiences (BR-RBAC-4)

Field-level rules `hideFrom` and `readOnlyFor` — renamed from `hideFromRoles`/`readOnlyForRoles` — live in field `config`. Each is a list drawn from the closed audience set:

`super_admin`, `admin`, `editor`, `contributor`, `viewer`, `end_user`, `anonymous`, `api_key`

- `hideFrom` strips the field on serialization for any audience in the list.
- `readOnlyFor` rejects the field at `content.Document.Set` for any audience in the list.

Semantics are unchanged from the prior naming; only the keys and the audience set changed — the audience set now spans the five roles plus `end_user`, `anonymous`, and `api_key`, closing the roles-only hole. Resolved field-rule state rides in `Decision.FieldRules` (§3 step 6).

## 6. API-Key Scope Shape (BR-AUTH-7, BR-API-2)

API-key access is defined solely by the key's scopes (§3 step 5) — not by intersection with the collection's grant matrix. The matrix has no `api_key` entry: the collection and the key are provisioned by the same super admin, so a second gate would add confusion, not security.

```json
{
  "collections": [
    { "id": "<uuid>", "actions": ["read", "create", "update", "delete"], "draftAccess": false }
  ],
  "passwordReset": false
}
```

- `collections[].id` references a collection by `id`, never `slug` — the same rule every collection reference follows (EC-4).
- `collections[].actions` is a subset of `read`, `create`, `update`, `delete`. There is no `publish` action in an API-key scope — publishing requires an admin principal (BR-LIFE-3).
- `collections[].draftAccess`: `true` includes drafts in reads for that collection; `false` (default) limits reads to published, non-trashed records (BR-API-2). This implements "unless the key scope explicitly grants draft access."
- `passwordReset`: a global capability, not per-collection. It gates the end-user password-reset request endpoint (BR-AUTH-13) and lives beside — not inside — the per-collection list because it carries no collection scope.
- A revoked key resolves to `anonymous` for every subsequent request (§3 step 5); it does not error at the transport layer — the request proceeds as if unauthenticated, subject to §1's `anonymous` grants.

## Rules Resolved Here

- **BR-RBAC-2** — §1 (grant-matrix schema; `Decision{Allowed, Predicate}` from the evaluator).
- **BR-RBAC-3** — §1, §3 step 1 (default-deny for governed classes; implicit `super_admin`/`admin` grant).
- **BR-RBAC-4** — §5 (field-rule audiences: `hideFrom`/`readOnlyFor`, closed audience set).
- **BR-RBAC-6** — §3 (row-scope predicates, e.g. `ownerOnly`, compile into every query via `Decision.Predicate`).
- **BR-RBAC-7** — §4 (closed grant-matrix schema; `422` on invalid writes; fail-closed evaluation at read, N-11).
