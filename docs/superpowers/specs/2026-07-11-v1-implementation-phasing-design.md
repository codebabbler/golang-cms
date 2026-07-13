# V1 Implementation Phasing — Design

**Date:** 2026-07-11 · **Status:** Approved (phase map + Phase 1) · **Owner:** Miraj Aryal

## Context

The documentation set is complete and approved: three principal-architect review rounds (round 3 verdict: APPROVED — READY FOR V1 IMPLEMENTATION PLANNING), zero open findings, core docs at v1.3. This spec fixes two things:

1. **The V1 phase map** — the decomposition of V1 into nine sub-projects, their scope boundaries, dependency order, and exit gates. This part is binding for sequencing.
2. **The Phase 1 specification** — implementation-grade detail for the first sub-project (the walking skeleton). Phases 2–9 get their own brainstorm → spec → plan cycle when they start, informed by the code that exists by then.

The architecture docs remain the sole normative source for behavior (authority chain in `CLAUDE.md`). This spec cites them; it does not restate or override them. Where implementation reveals a doc gap, the doc gets fixed in the same phase.

## Decisions

| ID | Decision | Chosen | Rationale |
|---|---|---|---|
| D-V1-1 | Phase granularity | Nine focused phases, one per subsystem seam | Plans stay reviewable; subagent tasks stay bite-sized; the `internal/` package boundaries (10-project-structure) already define the seams |
| D-V1-2 | Admin UI timing | Vite/go:embed pipeline proven in P1; screens deferred to P9 | Build pipeline de-risked on day one; UI built once against a frozen API; every backend phase is curl-smoke-testable without it |
| D-V1-3 | Spec depth | Full phase map at boundary level; only P1 at implementation grade. **Amended 2026-07-11 (user direction):** implementation plans for all nine phases are authored up front in `docs/superpowers/plans/v1/` (moved from the `plans/` root 2026-07-14 when plan directories split by version); because P2–P9 plans are written before their predecessors' code exists, each plan's execution opens with a re-validation pass against the then-current codebase, and the architecture docs win over any drifted plan detail (authority chain) | Sequencing locked while detail stays accurate (YAGNI); the up-front plans trade some drift risk for a complete reviewable programme, mitigated by the execution-time re-validation rule |
| D-V1-4 | Phase-map shape | Dependency backbone with early auth (option A) | No subsystem built twice; lifecycle complete before the public API reads it (roadmap's load-bearing-walls rule); admin auth at P2 makes every later phase exercisable with real credentials |
| D-P1-1 | Migration strategy | All 13 V1 system tables land in migration `0001` | `07-data-model.md` §System Tables is a thrice-reviewed closed inventory; one file is directly diffable against it; later phases add only queries + tests and never renumber; every `make test` run exercises the DDL from day one; later drift becomes an honest forward migration |
| D-P1-2 | BR trace gate during construction | `make trace` supports a checked-in waiver file of BR-IDs pending a named phase; the waiver shrinks each phase and must be empty at the V1 delivery gate | Without it the gate fails until P9 and gets ignored; with it the gate is green-and-meaningful from P1. The steady-state contract in `10-project-structure.md` (trace fails on uncovered non-structural rules) holds unchanged once the file is empty |
| D-P1-3 | Migration ledger | Custom runner (~50 lines) with a `cms_migrations` ledger table; `07-data-model.md` gains the table row in the same phase | Matches the documented model exactly (embedded, numbered, forward-only, advisory-locked) with no new dependency; the doc amendment follows the authority chain — docs get fixed, not contradicted |

## The V1 Phase Map

Each phase is one spec → plan → execution cycle. Exit gates are curl-verifiable plus `make test && make trace` green (CLAUDE.md's done-definition). Dependency edges: P2←P1, P3←P2, P4←P3, P5←P2, P6←P4+P5, P7←P4+P5+P6, P8←P7, P9←P6+P7+P8.

| # | Phase | Scope boundary | Exit gate |
|---|---|---|---|
| P1 | Foundation | Repo scaffold (go.mod, Makefile, sqlc.yaml, depguard), config loader (all seventeen env vars, list-all-missing, S3-unit atomicity per BR-MEDIA-6), startup steps 1/2/3/6 of BR-RUNTIME-3 + shutdown drain (EC-14), migration runner + migration `0001` (13 system tables), httpapi skeleton (envelope, `WriteError` + nine-code registry, RequestID/Logger/Recover middleware, compiled-in server timeouts), `/healthz` + `/readyz`, `audit.Recorder` interface + slog sink, Vite/Svelte shell + go:embed + cache busting (EC-15) | Boots against disposable PG; second process blocked by instance lock; SIGTERM drains clean; missing-env failure lists all; asset/index cache headers correct; toolchain green |
| P2 | Auth core | Argon2id + semaphore (BR-AUTH-3), `cms_users` queries, admin sessions (cookie + CSRF, sliding — BR-AUTH-1…5), setup token + `/setup` (BR-AUTH-11, startup step 5a), recovery + `/recover` (BR-AUTH-12, step 5b), RBAC role constants + require-role middleware, 4-hour re-auth window for destructive ops, auth-route rate limiting (limiter core + `CMS_TRUSTED_PROXY_CIDRS`, EC-10) | Smoke steps 1–2 via curl: fresh boot → setup token from warn log → first super admin → login/logout |
| P3 | Schema engine | `schema.Engine` (closed whitelist DDL — BR-SCHEMA-4, advisory key `0x636D7301`, `lock_timeout` 5 s / `statement_timeout` 60 s), `schema.Cache` atomic snapshot (BR-RUNTIME-4 bounds), 500-collection/200-field caps, field-type→DDL mapping, conversion matrix, default index set (status partial, `(created_by, id)`, `ix_` naming + truncation), collection/field admin CRUD, destructive gates (typed confirm + re-auth, BR-SCHEMA-7), startup step 4 goes real | UAC-1.1 via curl: create collection → `c_<slug>` exists with the BR-SCHEMA-8 anatomy; conversion matrix integration-tested |
| P4 | Content core | `query.Builder` (squirrel confined — BR-SCHEMA-3, all seven invariants), `content.Document.Set` (BR-RBAC-5, validation, 1 MiB field cap, Tiptap JSONB shape), lifecycle save/publish/trash/restore/purge + live-table/revisions contract (BR-LIFE-1…7, EC-5/EC-6), optimistic locking, admin records API incl. keyset cursors | UAC-1.4 via curl: stale `version` → 409 `conflict`; trash → restore round-trip; revisions append-only verified |
| P5 | Principals | API keys + scopes (BR-AUTH-6/7), `JWTService` (RS256, `cms_system_keys`, auto-generation, rotation — BR-AUTH-10), end-user register (BR-AUTH-14 gate) / login / refresh with rotation + reuse detection (BR-AUTH-8/9) / logout, password reset (BR-AUTH-13), `/api/admin/end-users` + disable (F-34), unified principal-resolution middleware | Refresh-reuse revokes the family; registration gate default-off; all four principal kinds resolve |
| P6 | Public API | `access.Evaluator` (grant matrix, `createStatus`, owner-draft — 12-access-rules, BR-API-2), `/api/v1` records CRUD, public filter grammar (BR-API-4), pagination caps, `count=exact` (authenticated only), one-level expand, ETag/304 + cache headers (BR-API-5), CORS (BR-API-6), rate limits (BR-API-7), idempotency keys (5 s wait, `request_hash`), per-route body caps | UAC-1.2 via curl: publish → unauthenticated fetch returns it; owner-draft visibility; cache-contract smoke check (09 runbook) passes |
| P7 | Media | `media.Service` presign→PUT→finalize→attach (BR-MEDIA-1…3), S3 SDK confinement + 10 s per-call timeout, media-less mode (BR-MEDIA-6 → 503 `unavailable`), deletion outbox enqueue + RESTRICT 409 (BR-MEDIA-5), media admin endpoints | UAC-1.5 against a disposable S3-compatible container; S3-less boot serves everything else |
| P8 | Jobs + bench | `jobs.Scheduler` + `Retention` (all eight duties of 07 §Retention), per-tick panic recovery, job telemetry (08), CI log-assertion tests (no-token-leak greps), `make bench` (seeded 100k rows, N-3/N-4) | Each retention duty integration-tested; bench meets N-3/N-4 |
| P9 | Admin UI | All screens per 06 (setup/login/recover, collection builder, records + Tiptap, media library, end-users, API keys, trash/revisions/compare), CSP headers, final embed; trace waiver file reaches empty | Full UAC-1.1…1.6 end-to-end through the UI — the V1 delivery gate (11-roadmap). Likely splits into two plans at its own planning time |

### Cross-phase rules

- **Audit call sites accrue per phase** against the P1 `audit.Recorder` interface; the closed action vocabulary grows with its subsystem (BR-AUDIT-1).
- **Configuration is parsed in full from P1** — all seventeen variables validate at startup from day one; later phases consume values, never reopen parsing.
- **Startup order is structural from P1** — steps 4 and 5 exist as seams that P3 and P2 fill; the fail-closed exits (N-11) are wired from the start.
- **Each phase updates the run-and-verify skill** where its smoke steps become real, and fixes any doc it falsifies (authority chain).
- **The trace waiver file (D-P1-2)** lists every non-structural BR-ID not yet covered, annotated with its owning phase; each phase's plan includes shrinking it. Empty file = V1 gate condition.
- **Git:** work lands on `main` per this project's convention (documentation history precedent); commits are plain messages without trailers.

## Phase 1 Specification — Foundation (walking skeleton)

**Deliverable:** one binary that boots per BR-RUNTIME-3 against PostgreSQL 16, migrates, holds the instance lock, serves `/healthz`, `/readyz`, and the embedded UI shell under the EC-15 cache contract, and drains per EC-14 — with `make build`, `make test`, `make trace`, `make generate`, and `make dev` all functional and green.

### Packages

| Package | Responsibility (P1 scope) |
|---|---|
| `cmd/cms` | `main.go`, flag-free, env-only; calls `app.Run` |
| `internal/app` | Config struct + validation (all seventeen variables), startup order, instance-lock watchdog, shutdown drain, wiring |
| `internal/httpapi` | chi router, envelope, `WriteError` + closed nine-code registry (BR-API-3), RequestID/Logger/Recover middleware, health endpoints, SPA serving |
| `internal/audit` | `Recorder` interface, `Event` type, slog sink (BR-AUDIT-1/2 shape per 08) |
| `internal/store` | Embedded migrations (`0001`), custom runner + `cms_migrations` ledger, sqlc scaffold with the first query |
| `web/` | Vite + Svelte 5 shell per 06 conventions — one placeholder route, no screens |

### Configuration (`internal/app`)

- Parses the full seventeen-variable table from `BUSINESS_RULES.md` § Naming Constants. Hard-required: `DATABASE_URL`, `CMS_MASTER_SECRET`. The `S3_*` group is optional as a unit — partial configuration fails startup naming the incomplete group (BR-MEDIA-6, startup half). Every remaining variable carries its documented default or is optional.
- Validation failure exits non-zero listing **every** missing/invalid variable at once (09 §Configuration).
- Values with no P1 consumer (e.g., `CMS_TRASH_RETENTION_DAYS`) still parse and validate — the struct is complete now.

### Startup and shutdown (`internal/app`)

Implements BR-RUNTIME-3 steps 1, 2, 3, 6; steps 4 and 5 are inert, clearly named seams:

1. Validate config; open the pgx pool (max 10 connections per 09 §Timeouts).
2. Run embedded migrations under advisory lock `0x636D7300`.
3. Acquire the instance lock (BR-RUNTIME-8): session-scoped `pg_advisory_lock` on `0x636D7302`, on a dedicated connection with TCP keepalives, retried with backoff for up to 120 s, then fail startup with a clear log line. After acquisition, a watchdog exits the process non-zero if the lock connection drops. The retry window and backoff are compiled-in constants; the constructor accepts them as parameters so integration tests can shorten them.
4. *Seam:* schema-cache load (P3). 5. *Seam:* setup/recovery token generation (P2).
6. Open the listener on `CMS_PORT`; `/readyz` begins returning 200.

Migration failure or instance-lock failure aborts startup non-zero (N-11). Shutdown per EC-14: flip `/readyz` and `/healthz` to 503 → stop tickers (seam — no tickers until P8) → `http.Server.Shutdown` with the 15 s window → exit 0; stragglers force-closed, logged and counted, exit 1 (BR-RUNTIME-6).

### Migration `0001` and store

- `0001_system_tables.sql` creates all 13 V1 system tables exactly per `07-data-model.md` §System Tables: `cms_users`, `cms_sessions`, `cms_api_keys`, `cms_end_users`, `cms_refresh_tokens`, `cms_system_keys`, `cms_collections`, `cms_fields`, `cms_revisions`, `cms_media`, `cms_reset_tokens`, `cms_idempotency_keys`, `cms_media_deletions` — including the documented uniques, partials, FKs, and CHECKs.
- Runner: embedded FS, files numbered `NNNN_description.sql`, forward-only, applied in order inside the advisory lock, recorded in `cms_migrations` (`version` PK, `name`, `applied_at`). Re-running is a no-op (idempotent per 09 §Backup restore-order note).
- Doc amendment in this phase: `07-data-model.md` gains the `cms_migrations` row (infrastructure ledger, written only by the runner).
- sqlc scaffold proves `make generate` with one real query: `CountUsers` (`SELECT count(*) FROM cms_users`) — the step-5 seam's future dependency (P2 consumes it for the BR-AUTH-11 emptiness check).

### httpapi skeleton

- Envelope (`{data, meta}`) and error shape (`{error: {code, message, details}}`) per 04; `WriteError` is the only error-writing path, over the closed nine-code registry (BR-API-3). Unmatched routes and methods answer through it per 04.
- Middleware: P1 implements the prefix of 04's normative chain — RequestID (UUIDv7, `X-Request-ID`, in-context) → Logger (one line per request, chi patterns not raw paths) → Recover (panic → `error` log with stack + request ID → `internal` envelope). RateLimit and Auth slot into their documented positions in P2/P5/P6; the order itself is a business-rule surface (04 §Middleware Order) and is structural from P1. The per-request 25 s context deadline and the 64 KiB default body cap (09 §Timeouts) wire in as server-level wrappers now; the 5 MiB record-write class arrives with its routes in P4.
- `http.Server` timeouts as compiled-in constants per 09 §Timeouts (ReadHeader 5 s, Read 30 s, Write 30 s, Idle 120 s).
- `/healthz`: 200 once started, 503 during drain. `/readyz`: DB ping, 200/503 (08 §Health).
- SPA serving from the embedded `web/dist`: hashed assets `Cache-Control: public, max-age=31536000, immutable`; `index.html` `Cache-Control: no-cache` (EC-15).
- Identity encoding only — compression belongs to the edge (09 §Reverse-Proxy Contract).

### audit

`Recorder` interface + `Event` (`action`, `actor_kind`, `actor_id`, `entity`, `detail`, `request_id`, `at`) + slog sink emitting the 08 §Audit line shape. The action vocabulary constant list starts empty-but-typed; subsystems append theirs in their phases. V2's DB sink slots behind the same interface.

### web shell

Vite + Svelte 5 scaffold per 06 conventions: one placeholder route, hashed asset output, `dist/` gitignored and embedded. Proves the `make build` ordering (vite → go, build fails on missing `web/dist`), go:embed, the EC-15 headers, and `make dev`'s proxy. No admin screens, no auth UI.

### Makefile and tooling

- Targets per 10-project-structure: `build`, `test` (unit + integration against a disposable PostgreSQL 16 container), `trace`, `generate`, `dev`. `bench` arrives in P8.
- `trace`: greps BR identifiers from `BUSINESS_RULES.md` against `_test.go` files; fails on any uncovered non-structural rule **not listed in the waiver file** (`docs/trace-waivers.txt`, one BR-ID + owning phase per line — D-P1-2). P1 ships the file pre-populated with every BR owned by P2–P9.
- `depguard` (or equivalent) configured now with the full 10-project-structure §Package Rules table — including rules for packages that don't exist yet, so the seams are enforced from each package's first commit.
- Generated sqlc code commits to the repository.

### Testing (N-10)

Integration (`//go:build integration`, disposable PG16): boot-order success path; missing-env aggregation; partial-S3 startup failure; second-instance lock rejection (shortened retry via constructor params); watchdog exit on lock-connection kill (`pg_terminate_backend`); SIGTERM drain with an in-flight request; migration idempotency on double-run. Unit: config parsing/defaults; `WriteError` registry closure (all nine codes, nothing else); envelope shapes; audit sink line shape; cache-header middleware.

BR-IDs leaving the waiver file in P1: BR-RUNTIME-3, BR-RUNTIME-6, BR-RUNTIME-8, BR-API-3, BR-AUDIT-1 (interface shape), BR-MEDIA-6 (startup half; the 503 half stays waived to P7).

### Out of scope for P1

Any auth or session logic; any `/api/v1` or `/api/admin` business route; schema cache internals; rate limiting; S3 client code; job tickers; admin screens. The step-4/5 seams and the empty audit vocabulary are the only footprints of later phases.

## Phase 1 Acceptance Criteria

1. `make build` produces the binary; deleting `web/dist` makes it fail.
2. With only `DATABASE_URL` + `CMS_MASTER_SECRET` set, the binary boots: `/healthz` and `/readyz` return 200; all 13 system tables + `cms_migrations` exist.
3. Unset both required variables → non-zero exit, output names both.
4. Set `S3_BUCKET` alone → startup fails listing the three missing `S3_*` variables (BR-MEDIA-6).
5. A second concurrent process fails startup after its bounded retry with a clear log line; killing the first process's lock connection makes it exit non-zero.
6. SIGTERM: `/readyz` flips 503, an in-flight request completes, exit 0.
7. Unknown route returns the 04 envelope via `WriteError`; the registry test proves exactly nine codes.
8. `curl -I /` shows `no-cache` on `index.html`; a hashed asset shows `immutable, max-age=31536000`.
9. `make test` green (unit + integration); `make trace` green with the waiver file listing only P2–P9 rules; `make generate` idempotent (no diff).
10. `depguard` config contains every 10-project-structure package rule; lint passes.
11. `07-data-model.md` contains the `cms_migrations` row (doc amendment committed with the phase).

## Handoff

Phase 1 proceeds to writing-plans → subagent-driven execution. Phases 2–9 each open with a short brainstorm → spec against the then-current codebase, using this document's phase map as the fixed sequencing contract. Changes to the phase map itself require revisiting this spec.
