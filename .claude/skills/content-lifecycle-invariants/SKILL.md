---
name: content-lifecycle-invariants
description: Use when touching internal/lifecycle — record saves, publish/unpublish, trash/restore/purge, revision handling, or retention. Encodes the write-transaction contract, publish semantics, restore drift rules, and retention exclusions.
---

# Content Lifecycle Invariants

Distilled from `docs/architecture/07-data-model.md`, `docs/architecture/03-dynamic-schema.md`, and `docs/BUSINESS_RULES.md` §3. Those documents are authoritative.

**Boundary:** this skill owns writes and state transitions. Read-side filtering (trash filter, scopes, predicates) belongs to `query-builder-invariants` — do not restate or reimplement it here.

## Hard Rules

1. **One write, one transaction** (BR-LIFE-1, BR-LIFE-7). Every create/update: live-row compare-and-set (`WHERE version = $expected`, mismatch → `409 conflict`, nothing written) + `cms_revisions` insert with the next monotonic `version_no` — in the same transaction. No code path writes one without the other.
2. **Publish is a copy, atomically** (BR-LIFE-2). Publish copies the chosen revision's `data` into the live row, sets `status='published'`, and moves the `published` flag to that revision in one transaction. The partial unique index guarantees at most one published revision per record — never work around it.
3. **The live row of a published record changes only via publish.** Edits to published records insert revisions only (the pending draft); the public API keeps serving the live row untouched until republish.
4. **Pending-draft detection is derived, never stored:** newest `version_no` > published revision's `version_no`. Do not add state columns for it.
5. **Restore appends** (BR-LIFE-1). `RestoreRevision` creates a new head revision from the old snapshot through the four drift-mapping rules of `docs/architecture/03-dynamic-schema.md` (same type → restore; changed type → safe cast else null; dropped field → ignore; new field → default else null). Skipped/defaulted fields go into the audit event `detail` (EC-5). History is never rewritten.
6. **Trash is a tombstone** (BR-LIFE-4, BR-LIFE-5). Trash sets `deleted_at`; restore clears it and the partial unique index decides collisions — a collision returns `409` naming the colliding field and record, and the row stays trashed (EC-6). Purge respects FK RESTRICT; blocked purges are skipped and logged, never forced.
7. **Retention never eats the load-bearing revisions** (BR-LIFE-8). Pruning (beyond `CMS_REVISION_LIMIT`, oldest first) skips the `published` revision and the newest revision, always.
8. **Every method emits audit** (BR-AUDIT-1) through `audit.Recorder` with the `domain.entity.verb` action vocabulary (`docs/architecture/08-observability.md`).
9. **Scheduled publishing (V2)** (BR-LIFE-9): `publish_at` ticker plus startup catch-up; each late publish logs at `warn` with scheduled vs. actual time (EC-13).

## Test Obligations

Trace to BR-LIFE-1…9 (`br-traceability-testing`). Highest-value adversarial tests: concurrent saves racing on `version`; restore into a unique collision; retention attempting to prune the published revision; a "publish" that tries to mutate the live row of a different record's transaction.

## Review Checklist

- [ ] Every write path pairs live-row CAS with a revision insert in one transaction?
- [ ] No new state column for anything derivable from `version_no` comparisons?
- [ ] Drift mapping applied on every restore, outcomes recorded in audit detail?
- [ ] 409s carry the colliding field name?
- [ ] Retention exclusions intact?
- [ ] `make trace` green for touched BR-LIFE rules?
