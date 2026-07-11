# Round-3 Gap Resolution — Design

**Date:** 2026-07-11 · **Status:** Approved (owner, 2026-07-11) · **Source:** `docs/reviews/architecture-review-round3-2026-07-11.md` (AR3-1…AR3-6 + open questions Q1–Q4)

Documentation-only pass. Quoted texts are normative and land verbatim (surrounding stitching may vary). Edited markdown docs bump to `**Version:** 1.3 · **Last Updated:** 2026-07-11`; `openapi.yaml` keeps `info.version: 1.0.0`.

## Decision Log (all recommended options accepted, 2026-07-11)

| ID | Decision | Resolves |
|---|---|---|
| D3-1 | Six AR3 fixes as recommended (incl. AR3-4 bounded-wait-then-409) | AR3-1…AR3-6 |
| D3-2 | Media-less boot: S3 group optional as a unit; new BR-MEDIA-6; new `unavailable` (503) error code; required set = `DATABASE_URL` + `CMS_MASTER_SECRET` | G-8 / Q3 |
| D3-3 | New N-14: external `/readyz` probe as availability SLI; closed alert list in 08 | G-5, G-6 / Q2 |
| D3-4 | Deployment assumptions hardened: dedicated database required; restart-on-exit supervisor required (systemd reference) | Q1, Q4 |

## 1. D3-1 — AR3 Fixes

**AR3-1 (07 §Indexes, append):**

> Every collection table also carries a `(created_by, id)` B-tree: the `ownerOnly` predicate (BR-RBAC-6) and owner-draft visibility (BR-API-2) both filter on `created_by`, and the owner-draft OR-shape plans as a bitmap-OR of this index and the status partial index.

03 needs no edit — its CreateCollection row defers to 07's default-index enumeration.

**AR3-2 (BR-API-6 + 04 §CORS, amend the preflight sentence):** replace "are handled before authentication and rate limiting and answered with" with:

> are handled before authentication and rate limiting — exempt because they carry no body, touch no database, and are edge-cacheable for 24 hours via `Access-Control-Max-Age` — and answered with

**AR3-3 (BR-MEDIA-5, append):**

> A retry that finds the object already gone (404) treats the deletion as complete and clears the queue row.

07 duty 8 becomes: "Retries `cms_media_deletions` entries older than 1 hour: delete the object — a 404 counts as success — then the row (BR-MEDIA-5)."

**AR3-4 (04 §Idempotent Creates, replace the blocking sentence):**

> A concurrent request with the same key waits on the unique index up to 5 seconds for the first transaction to resolve: if it committed, the second request returns the original outcome; if it aborted, the second proceeds as a fresh create; if it is still in flight after 5 seconds, the second returns `409 conflict` (request in progress — retry).

09's constants table gains: `| Idempotency duplicate-wait bound | 5 s |`

**AR3-5 (07, prose after the Collection Table Anatomy system-column table):**

> `created_by` holds the acting principal's UUID from whichever store issued it (`cms_users`, `cms_end_users`, or `cms_api_keys`) with no discriminator column. Display resolution checks `cms_users` → `cms_end_users` → `cms_api_keys`; predicate matching (`ownerOnly`, owner-draft) is raw UUID equality — cross-store collision is negligible.

**AR3-6 (09):** constants table gains `| S3 client per-call timeout (presign signing, HEAD, object delete) | 10 s |`; after the reverse-proxy contract's four-item list, one sentence:

> The edge also owns response compression (gzip/brotli); the binary serves identity encoding only.

## 2. D3-2 — Media-less Boot

**BR-MEDIA-6 (new, after BR-MEDIA-5, verbatim):**

> **BR-MEDIA-6.** The media subsystem is optional: when the four `S3_*` variables are entirely absent at startup, the binary boots in media-less mode — media routes (presign, finalize, delete) return `503` with code `unavailable` and everything else works. Partial configuration of the `S3_*` group fails startup listing the missing variables — half-configured storage is a mistake, not a mode. The `R2_*` variables remain conditional within media-enabled mode (`R2_ACCOUNT_ID` when using R2; `R2_PUBLIC_BUCKET_URL` for public delivery). Only `DATABASE_URL` and `CMS_MASTER_SECRET` are hard-required at startup.
> *Enforcement:* startup config validation + media handler gate.

- 04 error registry gains: `| `unavailable` | 503 | Media routes in media-less mode (BR-MEDIA-6). |` — the registry grows 8 → 9, an additive envelope change.
- `openapi.yaml` `Error.code` enum gains `unavailable`.
- BUSINESS_RULES env table, sentence after the table: "Hard-required: `DATABASE_URL`, `CMS_MASTER_SECRET`. The `S3_*` group is optional as a unit (BR-MEDIA-6), with the `R2_*` variables conditional within media-enabled mode; every remaining variable carries a default or is optional."
- 09 §Configuration, append: "Required at startup: `DATABASE_URL` and `CMS_MASTER_SECRET`; the `S3_*` group is optional as a unit (media-less mode, BR-MEDIA-6) — partial configuration of the group fails startup."
- `.claude/skills/run-and-verify/SKILL.md`, amend the S3-less sentence to cite the rule: "…everything except media flows works (media routes return `503 unavailable` — BR-MEDIA-6)…"

## 3. D3-3 — Availability SLI and Alerting

**N-14 (new, REQUIREMENTS §4, after N-13, verbatim):**

> **N-14.** Availability (N-13) is measured by an external HTTP probe against `/readyz` — any uptime monitor, evaluated monthly against the ~99.5% class. The probe and the alert wiring of `docs/architecture/08-observability.md` §Alerting are deployment obligations outside the binary; BR-RUNTIME-2 is untouched.

**08, new section before Edge-Case Coverage:**

> ## Alerting (N-14)
>
> The binary emits signals; wiring them to a notification channel is a deployment obligation. The closed V1 alert list:
>
> - Instance-lock-loss exit (BR-RUNTIME-8) — the process logged a lock-loss reason and exited.
> - Drain force-close count > 0 on shutdown (BR-RUNTIME-6).
> - A retention FK-RESTRICT skip naming the same record persisting beyond 7 days (BR-LIFE-8).
> - `/readyz` flapping, observed by the external availability probe (N-14).
> - (V2) Late scheduled publishes (BR-LIFE-9).

## 4. D3-4 — Deployment Assumptions Hardened (09)

Replace "golang-cms assumes it is the only user of its database's advisory-lock keyspace — deploy it into a dedicated database." with:

> golang-cms must be the only user of its database and its advisory-lock keyspace — a dedicated database is a hard deployment requirement.

Replace "the health endpoints make any supervisor (systemd `Restart=always` with `TimeoutStopSec=20`, or a container runtime honoring SIGTERM) sufficient." with:

> a restart-on-exit supervisor is required — the BR-RUNTIME-8 watchdog exits deliberately and relies on it. The reference is systemd `Restart=always` with `TimeoutStopSec=20`; any container runtime honoring SIGTERM and restarting on exit is equally supported.

## Identifier Delta

New: BR-MEDIA-6, N-14, error code `unavailable`. Amended: BR-API-6, BR-MEDIA-5. No new tables, columns, or env vars.

## Per-Document Edit Matrix

| File | Edits |
|---|---|
| `docs/BUSINESS_RULES.md` | BR-API-6 preflight clause; BR-MEDIA-5 404 clause; new BR-MEDIA-6; env-table required-set sentence; header 1.3. |
| `docs/REQUIREMENTS.md` | New N-14; header 1.3. |
| `docs/architecture/04-api-layer.md` | CORS preflight clause; idempotency 5 s wait; `unavailable` registry row; header 1.3. |
| `docs/architecture/07-data-model.md` | `(created_by, id)` index; duty-8 404 clause; `created_by` resolution prose; header 1.3. |
| `docs/architecture/08-observability.md` | Alerting section; header 1.3. |
| `docs/architecture/09-deployment.md` | Two constants rows; compression sentence; configuration required-set sentence; two hardened assumption sentences; header 1.3. |
| `docs/api/openapi.yaml` | `unavailable` in the Error code enum. |
| `.claude/skills/run-and-verify/SKILL.md` | BR-MEDIA-6 citation on the S3-less sentence. |
| `docs/reviews/architecture-review-round3-2026-07-11.md` | Append `## Resolution Status (2026-07-11)`: AR3-1…AR3-6 rows + one line noting Q1–Q4 resolved by D3-2/3/4. |

## Acceptance Criteria

1. `grep -n "created_by, id" docs/architecture/07-data-model.md` — present.
2. `grep -rn "BR-MEDIA-6" docs/BUSINESS_RULES.md docs/architecture/04-api-layer.md docs/architecture/09-deployment.md .claude/skills/run-and-verify/SKILL.md` — all four.
3. `grep -n "unavailable" docs/architecture/04-api-layer.md docs/api/openapi.yaml` — registry row + enum.
4. `grep -rn "N-14" docs/REQUIREMENTS.md docs/architecture/08-observability.md` — both.
5. `grep -n "5 seconds\|duplicate-wait" docs/architecture/04-api-layer.md docs/architecture/09-deployment.md` — wait rule + constants row.
6. `grep -rn "404" docs/BUSINESS_RULES.md docs/architecture/07-data-model.md | grep -i "success\|already gone"` — both clauses.
7. `grep -n "must be the only user\|restart-on-exit supervisor is required" docs/architecture/09-deployment.md` — both.
8. The six edited markdown docs read `**Version:** 1.3`; untouched docs unchanged.
9. Round-3 review ends with a Resolution Status table covering AR3-1…AR3-6 and Q1–Q4.
10. `grep -n "identity encoding" docs/architecture/09-deployment.md` — compression sentence present.
