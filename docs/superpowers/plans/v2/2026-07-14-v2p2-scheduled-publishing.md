# V2-P2 — Scheduled Publishing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Records publish automatically at `publish_at` within 60 seconds (BR-LIFE-9, F-26, UAC-2.3), missed schedules publish on the first tick after startup (EC-13), and the trace waiver file empties.

**Architecture:** `publish_at TIMESTAMPTZ NULL` joins the closed system-column set (BR-SCHEMA-8 amendment; migration 0004 backfills every existing `c_<slug>` table). `jobs.Publisher` is a thin 30 s ticker (first tick immediate) that delegates to `lifecycle.PublishDueScheduled`, which finds due records via a new `internal/query` helper (collection SQL stays inside `internal/query` — BR-SCHEMA-3) and publishes each through the existing `Publish` path as the new system principal. Comparisons use database `now()`, never the process clock (07).

**Tech Stack:** Go 1.25, pgx/v5, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v2-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code; docs win over drifted plan detail.
- SQL for collection tables exists only inside `internal/query` (BR-SCHEMA-3).
- DDL runs only through `schema.Engine`'s whitelist or the embedded migration ledger; this phase ships migration `0004`.
- Error responses only via `httpapi.WriteError` (BR-API-3). No new runtime deps; no new env vars.
- Every new mutation surface wires an audit call site and grows `audit.Actions()` in the same commit (spec B3).
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V2-3 — run before Task 1)

- [ ] Confirm: `lifecycle.NewService(pool, q, cache, rec, log)`; `(*Service).Publish(ctx, p access.Principal, col, recordID uuid.UUID, versionNo int64) error` with the `RoleEditor` floor (V1-P4 T5); `(*Service).Revisions(...)`; `jobs.NewScheduler(log)` + `Register(j Job, every, initialDelay time.Duration)` + `Job{Name() string; Run(ctx context.Context)}`-shaped interface (V1-P8 T1 — verify the exact Job method set); `schema.Cache` snapshot iteration; `access.Kind` constants (V1-P1 T2); the V2-P1 deliverables (`audit.Actions()`, fanout recorder).
- [ ] Confirm `0003` is the highest migration (V2-P1); renumber `0004` if not.
- [ ] Confirm the BR-SCHEMA-8 anatomy test name (`TestBR_SCHEMA_8_UAC_1_1_CreateCollectionAnatomy`, V1-P3 T?) and the system-column count it asserts (seven) — this phase changes it to eight.
- [ ] Confirm how `createTableDDL(slug, fields)` renders system columns (V1-P3 T3) and where the slug blocklist lives (`ValidateSlug`) — `publish_at` joins both.

## File Structure

```
internal/store/migrations/0004_publish_at.sql   backfill DO block + createTableDDL parity
internal/access/principal.go                    KindSystem + SystemPrincipal (modify)
internal/schema/ddl.go                          createTableDDL: publish_at column+index (modify)
internal/schema/validate.go                     blocklist += publish_at (modify)
internal/query/scheduled.go                     DueScheduled SQL helper
internal/lifecycle/schedule.go                  Schedule, Unschedule, PublishDueScheduled
internal/jobs/publisher.go                      30s ticker, first tick immediate
internal/httpapi/schedule_handlers.go           POST/DELETE …/schedule
web/src/routes/RecordEdit.svelte                schedule picker (modify)
web/src/routes/Records.svelte                   scheduled badge (modify)
web/e2e/v2p2-schedule.spec.ts                   UAC-2.3
docs/BUSINESS_RULES.md                          BR-SCHEMA-8 closed set +publish_at (amend)
docs/architecture/07-data-model.md              system-column table row (amend)
docs/architecture/08-observability.md           Publisher telemetry live (amend)
docs/trace-waivers.txt                          emptied
```

---

### Task 1: `publish_at` system column — migration 0004, DDL template, blocklist, BR/doc amendments

**Files:**
- Create: `internal/store/migrations/0004_publish_at.sql`
- Modify: `internal/schema/ddl.go` (`createTableDDL`), `internal/schema/validate.go` (blocklist), `docs/BUSINESS_RULES.md` (BR-SCHEMA-8), `docs/architecture/07-data-model.md` (system-column table)
- Test: extend the V1 anatomy test in `internal/schema/`

**Interfaces:**
- Produces: every `c_<slug>` table (existing and future) has `publish_at TIMESTAMPTZ NULL` + partial index `ix_<table>_publish_at ON (publish_at) WHERE publish_at IS NOT NULL`. The closed system-column set is now **eight**.

- [ ] **Step 1: Write the failing test** — extend `TestBR_SCHEMA_8_UAC_1_1_CreateCollectionAnatomy`: expected system-column set becomes `{id,status,version,created_at,updated_at,created_by,deleted_at,publish_at}` and the expected default index set gains `ix_c_<slug>_publish_at`. Add `TestMigration0004BackfillsExistingTables` (integration): create a collection on a database migrated only through `0003` (drive `store.Migrate` against a migrations FS filtered in the test, or create the collection then apply `0004` — whichever the V1 migrate API allows; re-validate), apply `0004`, assert the column and index exist on the pre-existing table.

- [ ] **Step 2: Run to verify it fails** — `./scripts/testdb.sh go test -tags integration ./internal/schema/ ./internal/store/` → FAIL (column count 7).

- [ ] **Step 3: Write migration 0004** (spec B5 backfill pattern):

```sql
-- 0004: publish_at joins the closed system-column set (BR-SCHEMA-8 amendment,
-- spec §D2). Backfills every existing collection table; createTableDDL adds the
-- column for tables created after this migration.
DO $$
DECLARE c RECORD;
BEGIN
  FOR c IN SELECT slug FROM cms_collections LOOP
    EXECUTE format('ALTER TABLE %I ADD COLUMN IF NOT EXISTS publish_at TIMESTAMPTZ', 'c_' || c.slug);
    EXECUTE format('CREATE INDEX IF NOT EXISTS %I ON %I (publish_at) WHERE publish_at IS NOT NULL',
                   'ix_c_' || c.slug || '_publish_at', 'c_' || c.slug);
  END LOOP;
END $$;
```

(Index-name truncation: if `ix_c_<slug>_publish_at` can exceed 63 bytes under the V1 slug cap, apply the V1 truncation helper's rule; re-validate the cap — if V1 caps slugs ≤ 40 chars this name always fits.)

- [ ] **Step 4: Extend `createTableDDL`** — the system-column block gains `publish_at TIMESTAMPTZ` and the default index set gains the partial index above. Add `publish_at` to the `ValidateSlug` blocklist (the "seven system columns" entry becomes eight — update the comment).

- [ ] **Step 5: Run to verify it passes** — integration suite green, including the V1 drift/rename suites (renames must carry the new index — the V1 rename path renames `ix_<table>_*` generically; verify).

- [ ] **Step 6: Amend the docs** — `BUSINESS_RULES.md` BR-SCHEMA-8: add `publish_at` to the enumerated closed set (wording: "…, `deleted_at`, and from V2, `publish_at`"). `07-data-model.md` §Anatomy: add row `| publish_at | TIMESTAMPTZ | NULL = unscheduled; non-NULL = publish the newest revision at that instant (BR-LIFE-9) |` and add the partial index to the index paragraph. Run `make trace` — still green (BR text change only; BR-LIFE-9 remains waived until Task 5).

- [ ] **Step 7: Commit** — `git commit -m "feat: publish_at system column — migration 0004, DDL template, BR-SCHEMA-8 amendment"`

---

### Task 2: System principal

**Files:**
- Modify: `internal/access/principal.go` (or wherever `Kind` constants live — re-validate)
- Test: extend the access package tests

**Interfaces:**
- Produces: `access.KindSystem Kind = "system"`; `access.SystemPrincipal() Principal` returning `Principal{Kind: KindSystem, ID: uuid.Nil, Role: RoleSuperAdmin}` — machine-initiated mutations pass every role floor and audit as `actor_kind="system"`, `actor_id NULL` (V2-P1 already stores Nil-UUID actors as NULL).

- [ ] **Step 1: Failing test** — `TestSystemPrincipalPassesRoleFloors`: `SystemPrincipal().Role.AtLeast(access.RoleEditor)` is true; `TestSystemPrincipalAuditsAsSystem` (in `internal/audit`, integration): emitting with `SystemPrincipal()` stores `actor_kind='system'`, `actor_id NULL`.
- [ ] **Step 2:** FAIL (undefined). **Step 3:** implement (constant + constructor, three lines each). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: system principal for machine-initiated mutations"`

---

### Task 3: Schedule/Unschedule lifecycle operations + admin endpoints

**Files:**
- Create: `internal/lifecycle/schedule.go`, `internal/httpapi/schedule_handlers.go`
- Modify: router registration; `internal/audit/actions.go` (+`content.record.schedule`, `content.record.unschedule`)
- Test: `internal/lifecycle/schedule_test.go`, `internal/httpapi/schedule_handlers_test.go`

**Interfaces:**
- Consumes: `lifecycle.Service` internals (same package), `query` update surface for system columns — **re-validate**: V1's `BuildUpdate` writes content columns; setting `publish_at` needs a dedicated `internal/query` helper `query.SetPublishAt(snap, col) (sql string)` compiling `UPDATE "c_<slug>" SET publish_at = $1, updated_at = now() WHERE id = $2 AND deleted_at IS NULL` (two variants or NULL-arg for clear).
- Produces:
  - `(*lifecycle.Service).Schedule(ctx, p access.Principal, col schema.Collection, recordID uuid.UUID, at time.Time) error` — floor `RoleEditor` (scheduling is deferred publishing — same right, BR-LIFE-3); `at` must be strictly future **by database `now()`** (query `SELECT now()` inside the same call; process clocks are not trusted — 07): else `ErrScheduleInPast`; trashed record → `ErrScheduleTrashed`; audit `content.record.schedule` with `detail: {publish_at}`.
  - `(*lifecycle.Service).Unschedule(ctx, p, col, recordID) error` — clears it; audit `content.record.unschedule`.
  - V1 `Publish` and `Trash` both gain one line: clear `publish_at` (spec §D2 "manual publish clears; trash clears").
  - `POST /api/admin/collections/{slug}/records/{id}/schedule` body `{"publish_at":"RFC3339"}` → 200 `{data:{publish_at}}`; past → 422 `validation_failed` naming `publish_at`; trashed → 422; `DELETE …/schedule` → 200. Both: `RequireSession`+`RequireCSRF`+editor floor via the service error mapping.

- [ ] **Step 1: Failing tests** — service-level: `TestScheduleRejectsPastByDBClock` (insert with `at = now()-1s` → `ErrScheduleInPast`), `TestScheduleRejectsTrashed`, `TestContributorCannotSchedule` (floor), `TestManualPublishClearsSchedule`, `TestTrashClearsSchedule`; handler-level integration: schedule → 200 and `publish_at` echoed; past → 422 naming `publish_at`; unschedule → 200 and NULL.
- [ ] **Step 2:** FAIL. **Step 3:** implement per the Interfaces block (the handler parses RFC 3339, maps `ErrScheduleInPast`/`ErrScheduleTrashed` → 422, floor error → 403). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: schedule/unschedule lifecycle operations (F-26)"`

---

### Task 4: Due-scan query helper + `PublishDueScheduled`

**Files:**
- Create: `internal/query/scheduled.go`, `internal/lifecycle/publish_due.go`
- Modify: `internal/audit/actions.go` — no new action (`content.record.publish` reused; the system actor distinguishes it)
- Test: `internal/query/scheduled_test.go`, `internal/lifecycle/publish_due_test.go`

**Interfaces:**
- Consumes: `schema.Cache` snapshot (collection list), `(*Service).Publish`, `(*Service).Revisions` (newest version), `access.SystemPrincipal()` (Task 2), `query.SetPublishAt` (Task 3).
- Produces:
  - `query.DueScheduled(col schema.Collection) string` — `SELECT id, publish_at FROM "c_<slug>" WHERE publish_at <= now() AND deleted_at IS NULL ORDER BY publish_at ASC LIMIT 100` (database `now()`; LIMIT bounds one tick's work — leftovers publish next tick 30 s later, still inside BR-LIFE-9's 60 s for realistic backlogs; the catch-up tick loops until the scan drains — see below).
  - `(*lifecycle.Service).PublishDueScheduled(ctx) (published, late int, err error)` — for each collection in the snapshot, scan; for each hit: publish the **newest revision** (max `version_no` via `Revisions` page 1) as `SystemPrincipal()`, clear `publish_at`, count `late++` and log `{"msg":"late scheduled publish","record_id":…,"late_by_ms":…}` at warn when `now()-publish_at > 60s` (BR-LIFE-9 telemetry, 08). Per-record failure logs at error, **retains** `publish_at` (next tick retries), continues. Loops the whole scan until a pass returns zero rows (catch-up after downtime drains fully on the first tick — EC-13).

- [ ] **Step 1: Failing tests** — `TestQueryDueScheduledShape` (unit: SQL string matches the contract; quoted identifier; no process-clock parameter); integration `TestBR_LIFE_9_DuePublishAndCatchUp`: seed record A scheduled 100 ms ahead and record B with `publish_at` one hour in the past (direct SQL in the test — simulating downtime), sleep 200 ms, call `PublishDueScheduled` once → both published (newest revision content on the live row, `status='published'`), both `publish_at` NULL, B counted late with the warn line present; `TestPublishDueRetainsScheduleOnFailure`: point one record at a revision-less state (or inject a failing publish via a canceled ctx on that iteration — re-validate the cleanest seam) → `publish_at` retained.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: due-schedule scan and system-actor publishing (BR-LIFE-9, EC-13)"`

---

### Task 5: `jobs.Publisher` ticker + waiver emptied

**Files:**
- Create: `internal/jobs/publisher.go`
- Modify: `internal/app/app.go` (register), `docs/trace-waivers.txt` (empty), `docs/architecture/08-observability.md`
- Test: `internal/jobs/publisher_test.go`

**Interfaces:**
- Consumes: `jobs.Scheduler.Register(j, 30*time.Second, 0)` — initialDelay 0 makes the first tick fire immediately after `Start`, which `app.Run` calls right after the listener opens (09 startup step 7).
- Produces: `jobs.NewPublisher(svc *lifecycle.Service, log *slog.Logger) *Publisher` implementing `jobs.Job` (`Name() == "publisher"`); each `Run` calls `PublishDueScheduled` and logs `job finish` with `published`, `late` counts (08 telemetry).

- [ ] **Step 1: Failing test** — `TestPublisherFirstTickRunsCatchUpBeforeFirstInterval` (integration): register on a scheduler, seed one past-due record, `Start(ctx)`, assert the record is published well before 30 s (poll ≤ 2 s); `TestPublisherTickPanicRecovered`: a publisher whose service panics (nil cache seam — re-validate) logs at error and the scheduler keeps ticking (V1-P8's recovery contract, asserted here for the new job).
- [ ] **Step 2:** FAIL. **Step 3:** implement (≈30 lines) + register in `app.Run` after the retention job. **Step 4:** PASS.
- [ ] **Step 5: Empty the waiver** — `docs/trace-waivers.txt`: delete the `BR-LIFE-9` line (file now has zero `BR-` lines). `make trace` → green — the `TestBR_LIFE_9_*` name from Task 4 satisfies it; a stale waiver would have failed.
- [ ] **Step 6: Amend 08** — mark `jobs.Publisher` live (present tense), name the counters (`published`, `late`) and the warn line `late scheduled publish`.
- [ ] **Step 7: Commit** — `git commit -m "feat: jobs.Publisher 30s ticker with startup catch-up; waiver file emptied"`

---

### Task 6: Admin UI — schedule picker + badge, e2e

**Files:**
- Modify: `web/src/routes/RecordEdit.svelte`, `web/src/routes/Records.svelte`
- Create: `web/e2e/v2p2-schedule.spec.ts`

- [ ] **Step 1: Failing e2e** — `v2p2-schedule.spec.ts` (UAC-2.3, practical timings): create a draft, open Schedule, pick a time 3 s ahead (`datetime-local` input + Schedule button), assert the list shows a "Scheduled" badge with the time; poll the public API until the record appears published (timeout 45 s — one tick); assert the badge cleared. Second scenario (catch-up half): via test API setup, set a past `publish_at` directly in the DB, restart the app container (`testenv.sh` helper — re-validate the e2e harness's restart affordance; if none exists, cover the restart half at the Go integration level only and note it in the spec run), assert published after boot.
- [ ] **Step 2: Implement** — RecordEdit: Schedule section (visible role ≥ editor, only when a publishable revision exists): datetime-local bound to `$state`, `api.post(...)/schedule`, Unschedule button when set; surface the 422 `publish_at` error inline via `FieldErrors`. Records list: badge column when `publish_at` non-null (list endpoint already returns system columns — re-validate; if not, extend the admin list projection to include `publish_at`).
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: schedule publishing UI (UAC-2.3)"`

---

### Task 7: Acceptance sweep

- [ ] `make test` green; `make trace` green with `grep -c '^BR-' docs/trace-waivers.txt` → `0` (spec §C3 — stays 0 for the rest of V2).
- [ ] V1 e2e suite + `v2p1-audit.spec.ts` + `v2p2-schedule.spec.ts` green.
- [ ] `make bench` unchanged (the partial index is write-path only; reads never touch `publish_at`).
- [ ] Audit stream shows `content.record.publish` events with `actor_kind='system'` for scheduled publishes and `content.record.schedule`/`unschedule` for the endpoints.
- [ ] Doc amendments landed: BR-SCHEMA-8 (eight columns), 07 anatomy row, 08 Publisher live.

## Self-Review Notes (execution-time attention)

- **Newest-revision choice** (spec §D2): scheduling publishes `max(version_no)` at fire time — content edited between scheduling and firing publishes the *latest* draft, not the draft as-of-scheduling. This matches manual publish's default and is the documented behavior; flag to the reviewer, not a bug.
- **Publish floor for the system actor** rides `RoleSuperAdmin` on `SystemPrincipal()` rather than a floor bypass — one lattice, no special cases.
- **LIMIT 100 per scan pass with drain loop**: bounds memory per pass while letting catch-up clear arbitrary backlogs on the first tick. A 100-record steady-state backlog per 30 s exceeds design load; if bench says otherwise, raise the limit — it is not a contract.
- **`SetPublishAt`/`DueScheduled` live in `internal/query`** purely for BR-SCHEMA-3 compliance; they are string builders, not part of the fluent Builder. Reviewers may prefer them as Builder methods — either home satisfies the BR as long as the package boundary holds.
