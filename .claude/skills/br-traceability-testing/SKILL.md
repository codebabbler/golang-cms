---
name: br-traceability-testing
description: Use when writing tests, changing BUSINESS_RULES.md, or making the trace CI job pass — encodes how tests bind to business-rule identifiers and which rules are exempt.
---

# BR Traceability Testing

Distilled from `docs/BUSINESS_RULES.md` § Rule-to-Code Traceability and `docs/architecture/10-project-structure.md`. Those documents are authoritative.

## The Contract

Every **non-structural** business rule maps to at least one Go test whose name embeds the rule identifier, hyphens as underscores:

```text
func TestBR_SCHEMA_7_DestructiveChangeRequiresRecentAuth(t *testing.T)
t.Run("BR-LIFE-5 restore collision returns 409", ...)
```

The CI job `make trace` extracts every `BR-` identifier from `docs/BUSINESS_RULES.md`, greps `_test.go` files for each, and fails on any uncovered non-structural rule.

## Structural Exemption

Rules tagged **[structural]** hold by construction — the enforcing path is the only path that exists (e.g., BR-MEDIA-1: no route accepts upload bytes; BR-RUNTIME-2: no other service client is wired). Review verifies them, not tests. Do not write vacuous tests for structural rules, and do not tag a rule [structural] to dodge writing a hard test — the tag belongs only where a test would assert the absence of code.

## Writing a Good BR Test

- Test the **invariant**, not the implementation: BR-LIFE-4 is "trashed records never appear outside the trash view" — assert on list/read/expand/search outputs, not on SQL strings.
- Prefer the adversarial case: BR-SCHEMA-3's best test feeds hostile identifiers (`posts"; DROP TABLE cms_users;--`); BR-AUTH-9's replays a rotated refresh token and asserts the whole family dies.
- One rule may need several tests (each enforcement surface); several rules must not share one test name — the trace grep needs each ID present.
- Integration tests requiring PostgreSQL take `//go:build integration` and run in `make test` against a disposable PG16 (`docs/architecture/10-project-structure.md`).

## When Rules Change

Changing BUSINESS_RULES.md and the code in one change set: update the rule text, the enforcement point, and the traced tests together — a rule whose test still asserts the old behavior is worse than an uncovered rule. New rules land with their tests in the same commit; `make trace` gates the merge either way.

## Review Checklist

- [ ] Every BR touched by this change has a test naming it?
- [ ] Adversarial case covered where the rule guards a boundary?
- [ ] No [structural] tag used to avoid a writable test?
- [ ] `make trace` and `make test` green?
