# Round-3 Gap Resolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Land the approved round-3 gap resolution (D3-1…D3-4, resolving AR3-1…AR3-6 and stakeholder questions Q1–Q4) — documentation only.

**Architecture:** Task 1 amends the authority document; Tasks 2–6 propagate; Task 7 appends the Resolution Status to the round-3 review and runs the acceptance sweep. Spec: `docs/superpowers/specs/2026-07-11-round3-gap-resolution-design.md`.

**Tech Stack:** Markdown/YAML edits; grep verification; git.

## Global Constraints

- **Documentation-only.** No files beyond the nine the tasks name.
- **Verbatim texts** land character-for-character; only stitching may vary.
- **Version stamps:** the six edited markdown docs (`BUSINESS_RULES`, `REQUIREMENTS`, `04`, `07`, `08`, `09`) become `**Version:** 1.3 · **Last Updated:** 2026-07-11`; `openapi.yaml` keeps `info.version: 1.0.0`; the skill file and review file carry no version header.
- **Git: commits authorized, plain messages, NO Co-Authored-By trailer.** Never stage `.claude/skills/system-design/` or `prompt.md`.
- The optional media unit is the **four `S3_*` vars**; `R2_*` stays conditional within media-enabled mode. The error registry grows to exactly nine codes.

---

### Task 1: BUSINESS_RULES.md

**Files:**
- Modify: `docs/BUSINESS_RULES.md`

- [ ] **Step 1: Pre-verify**

Run: `grep -n "handled before authentication and rate limiting and answered\|never strands an object\|CMS_REVISION_LIMIT" docs/BUSINESS_RULES.md`
Expected: all three present.

- [ ] **Step 2: BR-API-6 preflight rationale**

Replace `are handled before authentication and rate limiting and answered with` with:
```
are handled before authentication and rate limiting — exempt because they carry no body, touch no database, and are edge-cacheable for 24 hours via `Access-Control-Max-Age` — and answered with
```

- [ ] **Step 3: BR-MEDIA-5 404 clause**

Replace `so a crash between commit and object deletion never strands an object.` with:
```
so a crash between commit and object deletion never strands an object. A retry that finds the object already gone (404) treats the deletion as complete and clears the queue row.
```

- [ ] **Step 4: BR-MEDIA-6 (new, immediately after BR-MEDIA-5's enforcement line)**

```
- **BR-MEDIA-6.** The media subsystem is optional: when the four `S3_*` variables are entirely absent at startup, the binary boots in media-less mode — media routes (presign, finalize, delete) return `503` with code `unavailable` and everything else works. Partial configuration of the `S3_*` group fails startup listing the missing variables — half-configured storage is a mistake, not a mode. The `R2_*` variables remain conditional within media-enabled mode (`R2_ACCOUNT_ID` when using R2; `R2_PUBLIC_BUCKET_URL` for public delivery). Only `DATABASE_URL` and `CMS_MASTER_SECRET` are hard-required at startup.
  *Enforcement:* startup config validation + media handler gate.
```

- [ ] **Step 5: Env-table required-set sentence (new paragraph immediately after the table, before "## Edge-Case Coverage (Batch 1)")**

```
Hard-required: `DATABASE_URL`, `CMS_MASTER_SECRET`. The `S3_*` group is optional as a unit (BR-MEDIA-6), with the `R2_*` variables conditional within media-enabled mode; every remaining variable carries a default or is optional.
```

- [ ] **Step 6: Header 1.2 → 1.3; Post-verify**

Run: `grep -n "BR-MEDIA-6\|already gone\|edge-cacheable\|Hard-required" docs/BUSINESS_RULES.md && grep -c "Version:\*\* 1.3" docs/BUSINESS_RULES.md` → all present, count 1.

- [ ] **Step 7: Commit**

```bash
git add docs/BUSINESS_RULES.md
git commit -m "docs: BUSINESS_RULES round-3 gaps — BR-MEDIA-6, BR-MEDIA-5/BR-API-6 amendments, required-set note"
```

---

### Task 2: REQUIREMENTS.md — N-14

**Files:**
- Modify: `docs/REQUIREMENTS.md`

- [ ] **Step 1: Add N-14 (immediately after the N-13 line)**

```
- **N-14.** Availability (N-13) is measured by an external HTTP probe against `/readyz` — any uptime monitor, evaluated monthly against the ~99.5% class. The probe and the alert wiring of `docs/architecture/08-observability.md` §Alerting are deployment obligations outside the binary; BR-RUNTIME-2 is untouched.
```

- [ ] **Step 2: Header 1.2 → 1.3; Post-verify**

Run: `grep -n "N-14" docs/REQUIREMENTS.md` → present.

- [ ] **Step 3: Commit**

```bash
git add docs/REQUIREMENTS.md
git commit -m "docs: REQUIREMENTS — N-14 availability SLI via external readyz probe"
```

---

### Task 3: 04-api-layer.md

**Files:**
- Modify: `docs/architecture/04-api-layer.md`

- [ ] **Step 1: CORS preflight rationale (same replacement as Task 1 Step 2, applied to the §CORS paragraph)**

Replace `are handled before authentication and rate limiting and answered with` with:
```
are handled before authentication and rate limiting — exempt because they carry no body, touch no database, and are edge-cacheable for 24 hours via `Access-Control-Max-Age` — and answered with
```

- [ ] **Step 2: Idempotency 5-second wait**

Replace:
```
A concurrent request with the same key blocks on the unique index until the first transaction resolves: if it committed, the second request returns the original outcome; if it aborted, the second proceeds as a fresh create.
```
with:
```
A concurrent request with the same key waits on the unique index up to 5 seconds for the first transaction to resolve: if it committed, the second request returns the original outcome; if it aborted, the second proceeds as a fresh create; if it is still in flight after 5 seconds, the second returns `409 conflict` (request in progress — retry).
```

- [ ] **Step 3: `unavailable` registry row (after the `internal` row of the error-code table)**

```
| `unavailable` | 503 | Media routes in media-less mode (BR-MEDIA-6). |
```

- [ ] **Step 4: Header 1.2 → 1.3; Post-verify**

Run: `grep -n "unavailable\|5 seconds\|edge-cacheable" docs/architecture/04-api-layer.md` → all present.

- [ ] **Step 5: Commit**

```bash
git add docs/architecture/04-api-layer.md
git commit -m "docs: API layer — unavailable code, idempotency wait bound, preflight rationale"
```

---

### Task 4: 07-data-model.md

**Files:**
- Modify: `docs/architecture/07-data-model.md`

- [ ] **Step 1: `(created_by, id)` index (append to the **Indexes.** paragraph)**

```
 Every collection table also carries a `(created_by, id)` B-tree: the `ownerOnly` predicate (BR-RBAC-6) and owner-draft visibility (BR-API-2) both filter on `created_by`, and the owner-draft OR-shape plans as a bitmap-OR of this index and the status partial index.
```

- [ ] **Step 2: Duty-8 404 clause**

Replace `8. Retries `cms_media_deletions` entries older than 1 hour: delete the object, then the row (BR-MEDIA-5).` with:
```
8. Retries `cms_media_deletions` entries older than 1 hour: delete the object — a 404 counts as success — then the row (BR-MEDIA-5).
```

- [ ] **Step 3: `created_by` resolution prose (new paragraph between the system-column table and the "User-field storage follows..." paragraph)**

```
`created_by` holds the acting principal's UUID from whichever store issued it (`cms_users`, `cms_end_users`, or `cms_api_keys`) with no discriminator column. Display resolution checks `cms_users` → `cms_end_users` → `cms_api_keys`; predicate matching (`ownerOnly`, owner-draft) is raw UUID equality — cross-store collision is negligible.
```

- [ ] **Step 4: Header 1.2 → 1.3; Post-verify**

Run: `grep -n "created_by, id\|404 counts as success\|whichever store" docs/architecture/07-data-model.md` → all present.

- [ ] **Step 5: Commit**

```bash
git add docs/architecture/07-data-model.md
git commit -m "docs: data model — created_by index and resolution order, outbox 404 semantics"
```

---

### Task 5: 08-observability.md + 09-deployment.md

**Files:**
- Modify: `docs/architecture/08-observability.md`
- Modify: `docs/architecture/09-deployment.md`

- [ ] **Step 1: 08 — Alerting section (new, immediately before "## Edge-Case Coverage (this document)")**

```
## Alerting (N-14)

The binary emits signals; wiring them to a notification channel is a deployment obligation. The closed V1 alert list:

- Instance-lock-loss exit (BR-RUNTIME-8) — the process logged a lock-loss reason and exited.
- Drain force-close count > 0 on shutdown (BR-RUNTIME-6).
- A retention FK-RESTRICT skip naming the same record persisting beyond 7 days (BR-LIFE-8).
- `/readyz` flapping, observed by the external availability probe (N-14).
- (V2) Late scheduled publishes (BR-LIFE-9).
```

- [ ] **Step 2: 09 — two constants rows (after `| pgx pool max connections | 10 |`)**

```
| Idempotency duplicate-wait bound | 5 s |
| S3 client per-call timeout (presign signing, HEAD, object delete) | 10 s |
```

- [ ] **Step 3: 09 — compression sentence (new paragraph immediately after the reverse-proxy contract's item 4, before "## SPA Cache Busting")**

```
The edge also owns response compression (gzip/brotli); the binary serves identity encoding only.
```

- [ ] **Step 4: 09 — configuration required-set sentence**

Replace `The binary validates required variables at startup and exits non-zero listing every missing one at once, not the first.` with:
```
The binary validates required variables at startup and exits non-zero listing every missing one at once, not the first. Required at startup: `DATABASE_URL` and `CMS_MASTER_SECRET`; the `S3_*` group is optional as a unit (media-less mode, BR-MEDIA-6) — partial configuration of the group fails startup.
```

- [ ] **Step 5: 09 — hardened assumptions**

Replace `golang-cms assumes it is the only user of its database's advisory-lock keyspace — deploy it into a dedicated database.` with:
```
golang-cms must be the only user of its database and its advisory-lock keyspace — a dedicated database is a hard deployment requirement.
```
Replace `the health endpoints make any supervisor (systemd `Restart=always` with `TimeoutStopSec=20`, or a container runtime honoring SIGTERM) sufficient.` with:
```
a restart-on-exit supervisor is required — the BR-RUNTIME-8 watchdog exits deliberately and relies on it. The reference is systemd `Restart=always` with `TimeoutStopSec=20`; any container runtime honoring SIGTERM and restarting on exit is equally supported.
```

- [ ] **Step 6: Both headers 1.2 → 1.3; Post-verify**

Run: `grep -n "## Alerting (N-14)" docs/architecture/08-observability.md && grep -n "duplicate-wait\|identity encoding\|must be the only user\|restart-on-exit supervisor is required\|media-less mode" docs/architecture/09-deployment.md` → all present.

- [ ] **Step 7: Commit**

```bash
git add docs/architecture/08-observability.md docs/architecture/09-deployment.md
git commit -m "docs: observability + deployment — alert list, constants, hardened deploy requirements"
```

---

### Task 6: openapi.yaml + run-and-verify skill

**Files:**
- Modify: `docs/api/openapi.yaml`
- Modify: `.claude/skills/run-and-verify/SKILL.md`

- [ ] **Step 1: openapi — enum entry (after `- internal` in the Error code enum)**

```
                - unavailable
```
(Indentation matches the sibling enum entries.)

- [ ] **Step 2: Skill — BR-MEDIA-6 citation**

Replace `Without the `S3_*`/`R2_*` variables the binary boots and everything except media flows works — media smoke steps below are skipped in that case.` with:
```
Without the `S3_*` variables the binary boots in media-less mode (media routes return `503 unavailable` — BR-MEDIA-6) and everything except media flows works — media smoke steps below are skipped in that case.
```

- [ ] **Step 3: Post-verify**

Run: `grep -n "unavailable" docs/api/openapi.yaml .claude/skills/run-and-verify/SKILL.md && python3 -c "import yaml; yaml.safe_load(open('docs/api/openapi.yaml')); print('YAML valid')"` → both present, YAML valid.

- [ ] **Step 4: Commit**

```bash
git add docs/api/openapi.yaml .claude/skills/run-and-verify/SKILL.md
git commit -m "docs: openapi + run-and-verify — unavailable code, media-less mode citation"
```

---

### Task 7: Round-3 review Resolution Status + acceptance sweep

**Files:**
- Modify: `docs/reviews/architecture-review-round3-2026-07-11.md`

- [ ] **Step 1: Replace the closing line and append the table**

Replace `*End of Round-3 review. Findings AR3-1…AR3-6; disposition tracking should follow the established Resolution Status convention when addressed.*` with:

```
*End of Round-3 review. Findings AR3-1…AR3-6; dispositions below.*

## Resolution Status (2026-07-11)

Dispositioned against `docs/superpowers/specs/2026-07-11-round3-gap-resolution-design.md` (owner-approved, D3-1…D3-4); all edits committed. Stakeholder questions Q1–Q4 (Section 19) are resolved by D3-2 (media-less boot, BR-MEDIA-6), D3-3 (N-14 SLI + 08 §Alerting), and D3-4 (dedicated database and restart-on-exit supervisor as hard requirements).

| AR3 # | Sev | Finding | Disposition |
|---|---|---|---|
| AR3-1 | Medium | No `created_by` index despite ownerOnly + owner-draft predicates. | Resolved: `(created_by, id)` B-tree added to the default index set (07 §Indexes). |
| AR3-2 | Low | Preflight rate-limit exemption stated without rationale. | Resolved: rationale clause in BR-API-6 and 04 §CORS. |
| AR3-3 | Low | Outbox retry vs already-deleted object unspecified. | Resolved: 404-counts-as-success in BR-MEDIA-5 and 07 duty 8. |
| AR3-4 | Low | Idempotency duplicates hold a pool connection to the deadline. | Resolved: 5 s wait-then-409 in 04; bound pinned in 09's constants. |
| AR3-5 | Low | `created_by` cross-store resolution order undocumented. | Resolved: lookup-order paragraph in 07 §Collection Table Anatomy. |
| AR3-6 | Low | Storage-call timeouts unpinned; compression unstated. | Resolved: 10 s S3 per-call row + identity-encoding sentence in 09. |

All round-3 findings are resolved in the committed documentation set as of 2026-07-11.
```

- [ ] **Step 2: Acceptance sweep (all 10 spec criteria)**

```bash
grep -n "created_by, id" docs/architecture/07-data-model.md
grep -rln "BR-MEDIA-6" docs/BUSINESS_RULES.md docs/architecture/04-api-layer.md docs/architecture/09-deployment.md .claude/skills/run-and-verify/SKILL.md
grep -n "unavailable" docs/architecture/04-api-layer.md docs/api/openapi.yaml
grep -rln "N-14" docs/REQUIREMENTS.md docs/architecture/08-observability.md
grep -n "5 seconds\|duplicate-wait" docs/architecture/04-api-layer.md docs/architecture/09-deployment.md
grep -rn "counts as success\|already gone" docs/BUSINESS_RULES.md docs/architecture/07-data-model.md
grep -n "must be the only user\|restart-on-exit supervisor is required" docs/architecture/09-deployment.md
grep -c "Version:\*\* 1.3" docs/BUSINESS_RULES.md docs/REQUIREMENTS.md docs/architecture/04-api-layer.md docs/architecture/07-data-model.md docs/architecture/08-observability.md docs/architecture/09-deployment.md
tail -3 docs/reviews/architecture-review-round3-2026-07-11.md
grep -n "identity encoding" docs/architecture/09-deployment.md
```
Expected: every grep non-empty; the version count is 1 per file. Fix any mechanical miss; report anything structural.

- [ ] **Step 3: Commit**

```bash
git add docs/reviews/architecture-review-round3-2026-07-11.md
git commit -m "docs: round-3 review — Resolution Status, AR3-1..AR3-6 and Q1-Q4 dispositioned"
```

---

## Self-Review (done at write time)

- **Spec coverage:** D3-1 → Tasks 1/3/4/5; D3-2 → Tasks 1/3/5/6; D3-3 → Tasks 2/5; D3-4 → Task 5; disposition → Task 7. Every edit-matrix row has a task.
- **Placeholder scan:** none.
- **Consistency:** `unavailable`, `BR-MEDIA-6`, `N-14`, `5 s`/`5 seconds`, `10 s`, `(created_by, id)` spelled identically across tasks; version target 1.3 uniform.
