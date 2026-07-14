# Plan Review — V1–V3 Implementation Plans (22 plans)

**Date:** 2026-07-14 · **Reviewer:** inline single-context review (user-directed after the subagent wave hit session limits), full-text pass over every plan with claim-level verification against `BUSINESS_RULES.md`, `docs/architecture/*`, and REQUIREMENTS — evidence and per-plan verified-good ledgers live in `plan-review-handoff.md` (same directory). Rubric: spec fidelity, doc conformance (authority chain), internal coherence, edge cases, project-skill invariants, plus cross-cutting passes (interface continuity, harness arithmetic, BR/UAC coverage, EC register, security, ops).

**Verdict:** The corpus is structurally sound — interface continuity holds across all 22 plans, waiver/migration arithmetic is exact, every EC and UAC lands in a named task. But 10 HIGH findings would produce wrong or gate-breaking implementations if executed as written, and 20 MEDIUMs are plausible implementer traps. All are line-item fixable; none invalidates the phasing or requires re-planning a phase.

## HIGH (fix before execution)

| ID | Where | Defect | Proposed fix |
|---|---|---|---|
| H1 (F-v1p1-1) | v1p1 T3 | `scripts/testdb.sh` hardcodes `docker`; the dev machine is podman-only (documented constraint) — `make test` unrunnable. v1p7 even *assumes* the detection exists. | Runtime fallback `CONTAINER_ENGINE`→docker→podman in testdb.sh (and testenv.sh inherits). |
| H2 (F-v1p1-3) | v1p1 T3 / 07 | `cms_api_keys.created_by` FK (07's own row) makes any admin who ever created a key **undeletable**, contradicting 07's admin-deletion contract (revoked keys are kept forever — BR-AUTH-7). | Drop the FK (created_by is FK-less everywhere else) + one-line 07 amendment. Doc conflict — needs your call. |
| H3 (F-v1p1-4 + F-v2p1-1) | v1p1 T9 / v2p1 T6 | trace.sh waiver regex `^${id}[[:space:]]` never matches v2p1's planned bare `BR-LIFE-9` reseed → `make trace` red at the v2p1 gate. | Regex `^${id}([[:space:]]|$)` AND write the annotated form `BR-LIFE-9 V2-P2`. |
| H4 (F-v1p3-1) | v1p3 T5 / 03 | `pg_advisory_xact_lock` releases AT commit, but 03 §88/BR-RUNTIME-7 require the cache swap *before* lock release — unsatisfiable; concurrent Applies can swap stale snapshots. 03 is self-contradictory. | In-process mutex around `Apply` (sound under single-instance BR-RUNTIME-8) + 03 amendment naming the mechanism. |
| H5 (F-v1p3-4) | v1p3 / 03 | No whitelisted op provisions indexes for fields added after collection creation: `AddField` is bare ADD COLUMN, `AddIndex` only does the composite form, partial-unique has **no op** — late `unique:true` fields get zero enforcement; schema import (v2p8) hits it wholesale. | `AddField` emits column + its declared index DDL (composite/partial-unique/trgm) in one tx; 03 template-row amendment. |
| H6 (F-v1p4-1) | v1p4 T4 | Revision merge base is the live row — for a published record with a pending draft, a second partial update merges against *published* values, silently discarding earlier draft edits. | Merge base = newest revision when `max(version_no) > published`; regression test (partial-update-twice-on-published). Also carries `$seo` (v2p3) and M2M arrays (v2p7). |
| H7 (F-v1p5-1) | v1p5 T4 / 05 §5 | Register: duplicate email → 422 vs success → 201-with-tokens — a response-shape oracle; 05 explicitly forbids register disclosing account existence. | Uniform `201 {data:{}}` for success AND duplicate (drop auto-login); client logs in after. Ripples: v1p9 flow, openapi. |
| H8 (F-v1p8-1) | v1p8 T3 / BR-AUDIT-1 | BR-AUDIT-1 enumerates **purge** as audited; retention's trash purges emit nothing, and V1 has no system actor at all. | Pull `KindSystem`+`SystemPrincipal()` forward from v2p2 into v1p8 (or P1); duty 1 emits `content.record.purge`; v2p2 becomes consumer. |
| H9 (F-v2p8-1) | v2p8 T4 | Erasure transaction deletes `cms_end_users` FIRST while `cms_refresh_tokens.end_user_id` FK still references it → **erasure always fails** for users with tokens. | Reorder: children first, user row last (existence check up front). |
| H10 (F-v3p3-1) | v3p3 T4 | `checkout.session.completed → paid` unconditionally; async payment methods fire `completed` with `payment_status:"unpaid"` → **order marked paid, money never received** (later `async_payment_failed` no-ops). | Check `payment_status` on completed; add `async_payment_succeeded → paid`; optionally pin `payment_method_types=card`. |

## MEDIUM (fix or explicitly accept at triage)

1. **M1 (v1p2-1 + v1p9-1)** CSRF rotation on every `GET /csrf` breaks multi-tab sessions; fix pair: `csrf_mismatch` reason detail + client single-retry-with-refetch.
2. **M2 (v1p4-2)** 5 MiB body-cap class applied only to record create — `PUT` update (large richText edits) 413s at the 64 KiB default.
3. **M3 (v1p4-3)** Two conflicting `content.Set` signatures stated in one task (deliberation text left in); keep the final one.
4. **M4 (v1p5-2)** `APIKeyService.Create` omits `createdBy` (NOT NULL column).
5. **M5 (v1p5-3 + v1p9-2)** Role-management UI is promised (v1p9 T9) with **no role-update endpoint anywhere**; admin deletion contracted by 07 also unplanned. Add `PUT /users/{id}/role` (+ last-super_admin guard); deletion = your call with H2.
6. **M6 (v1p7-1)** Media presign/finalize floored at editor+ — contributors can never upload; floor at contributor+.
7. **M7 (v1p8-2)** Bench fires ~600 anonymous reads/min into BR-API-7's 300/min limiter → 429s mid-bench; authenticate bench requests.
8. **M8 (v1x-1)** REQUIREMENTS defines UAC-1.1…**1.7**; roadmap gate row, V1 spec, v1p9, and every later "V1 gates green" say 1.1…1.6. Enumeration fix across four docs/plans (content already covered).
9. **M9 (v2p2-1)** Migration 0004's DO-block index name exceeds 63 bytes at V1's 55-char slug cap → silent PG truncation diverging from `indexName()`; implement the truncation rule in plpgsql + parity test.
10. **M10 (v2p3-1)** Reserved `seo` key writable by end_user/api_key principals (SEO spam surface on createStatus-published flows); gate to admin kinds.
11. **M11 (v2p5-1 + v3p1-1)** Field **rename** strands slugs stored in `search_config` and `commerce_config`; one rename-propagation hook for both.
12. **M12 (v2p5-2)** Generated tsvector can exceed the 1 MiB tsvector limit (16 × 1 MiB richText) → row writes fail on searched collections; cap extractor output in the expression.
13. **M13 (v2p6-1)** Delivery claim semantics: FOR UPDATE SKIP LOCKED without a held tx claims nothing; state at-least-once delivery (dedupe on `X-CMS-Delivery`) and drop the misleading clause.
14. **M14 (v2p6-2)** Unpublish emits no webhook event while the 04 amendment invites TTL-lengthening → indefinite stale cache on unpublish. Add `record.unpublish` (F-22 touch) or scope the TTL advice — your call.
15. **M15 (v2p7-1)** Publish/restore M2M reconciliation hard-fails on purged targets (FK 23503); filter-and-report like drift's `skipped`.
16. **M16 (v2p8-2)** Revision redaction validates against the *current* snapshot — PII under renamed/dropped slugs is unredactable; validate shape-only.
17. **M17 (v3p2-1)** Cart read-modify-write has no concurrency control (lost updates); `FOR UPDATE` tx.
18. **M18 (v3p4-1)** Paywall depends on CF `requireSignedURLs` being enabled — operator prerequisite missing from docs and the manual gate step.
19. **M19 (v3p4-2)** Unpublishing a purchased record locks out entitled buyers (visibility gate precedes entitlement) — accept+document or bypass — product decision.
20. **M20 (v2p8-3)** Schema import loses indexes on imported fields — resolved automatically by H5.

## LOW (~25 — batch-fixable)

Full list with evidence in `plan-review-handoff.md`. Themes: deliberation text left in two plans (v1p4/v1p5); minor citation/artifact nits ("V1-P3 T?", "(V3)" preamble grep, 17+13 typo); SPA index ETag; session-table touch frequency; missing secondary indexes; renderer mark-allowlist vs starter-kit (`code`/`strike`); XSS corpus case/whitespace scheme variants; CGNAT range in SSRF denylist; consumer replay-window guidance; self-referencing M2M test; block-in-expanded-target test; ParsePagination principal-signature growth; media-less "every method" wording; cart display vs hideFrom; system-actor note staleness (v2p1) after H8.

## What held up (highlights)

Waiver arithmetic exact (48 = 62 − 8 − 4 − 2; per-phase shrinks sum); migrations 0001–0012 gapless; EC-1…16 all placed; interface hand-offs verified across all 22 plans (one signature drift, LOW); security posture strong except H7 (never-log discipline, HMAC/SSRF/token handling all verified); the re-validation preambles correctly anticipate most drift; three findings are self-detecting at execution (H3's gate, v2p5's generated-column assumption, H9's own test) — the plans' TDD discipline is doing its job.

## Proposed next step

Per the approved flow: you triage (numbers are enough — e.g. "fix all HIGH + M1-M20 except M14/M19 accept-as-documented"); I apply the fixes to the plans/specs/docs and re-review only the changed documents.
