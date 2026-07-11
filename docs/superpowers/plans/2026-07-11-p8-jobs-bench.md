# P8 Jobs + Bench Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Background work and the performance gate: `jobs.Scheduler` + the eight-duty `jobs.Retention` ticker with per-tick panic recovery and telemetry, the no-secret-leak log-assertion tests, and `make bench` asserting N-3/N-4 on a seeded 100k-row database.

**Architecture:** Implements 08 §Job Telemetry, 07 §Retention (BR-LIFE-8), BR-RUNTIME-5, and the EC-14 shutdown step 2 (tickers stop before drain). The Publisher ticker is V2 — only its interface slot exists.

**Tech Stack:** stdlib tickers/goroutines only (BR-RUNTIME-5); pgx `CopyFrom` for bench seeding.

> **Authored before execution (D-V1-3 amendment):** re-validate at execution start; docs win. Invoke `content-lifecycle-invariants` (retention exclusions), `sqlc-workflow`, and `br-traceability-testing` before their tasks.

## Global Constraints

- Background work = in-process goroutine tickers, nothing external (BR-RUNTIME-5). Retention ticks **hourly**; the first tick runs shortly after startup (small fixed delay, e.g. 1 min — compiled constant) so a long-running instance is never a precondition for cleanup.
- Every tick wraps in panic recovery: a panicking duty logs at `error` and the ticker skips to the next tick (08 §Job Telemetry). Tickers log start/finish with per-duty counts and durations.
- The eight duties of 07 §Retention verbatim, in order:
  1. Trash purge past `CMS_TRASH_RETENTION_DAYS` — FK RESTRICT honored; each skip logs at `warn` naming the blocking reference (the BR-LIFE-8 alert input, 08 §Alerting).
  2. Revision pruning beyond `CMS_REVISION_LIMIT` per record, **oldest first, never the `published` revision, never the newest**.
  3. `cms_media` rows `pending` > 24 h: storage object deleted first (when present), then the row (BR-MEDIA-2).
  4. `cms_sessions` past idle (7 d) / absolute (30 d) expiry.
  5. `cms_reset_tokens` used or expired.
  6. `cms_refresh_tokens` rotated-or-revoked and older than 30 days.
  7. `cms_idempotency_keys` older than 24 h.
  8. `cms_media_deletions` older than 1 h: delete object — 404 counts as success — then the row (BR-MEDIA-5).
- Shutdown order goes fully live (EC-14): flip health → **stop tickers** → drain → exit.
- Bench (N-3/N-4): seeded **100,000 rows** in one collection (text title indexed, number rank indexed, richText body); p95 of ≥ 200 published single-record fetches ≤ **50 ms**; p95 of ≥ 200 list queries (limit 25, indexed sort) ≤ **120 ms** — measured through the full HTTP stack. Thresholds bind on the 2 vCPU / 4 GB reference host; the harness prints its numbers and asserts, with an env escape (`BENCH_ASSERT=0`) for non-reference dev machines — CI/release runs assert.
- Log assertions (08): across a captured full-flow log, no `cms_`-prefixed token value, no `Set-Cookie` value, no JWT (`eyJ` triple-segment), no presigned URL query material (`X-Amz-Signature`) may appear — the two BR-AUTH-11/12 setup/recovery warn lines are the sole exceptions.
- Waiver shrink: `BR-LIFE-8`, `BR-RUNTIME-5`.
- `WriteError` only (no HTTP surface added here); commits plain; branch `main`.

## File Structure

```
internal/jobs/scheduler.go            Scheduler: register/start/stop tickers (BR-RUNTIME-5)
internal/jobs/scheduler_test.go
internal/jobs/retention.go            the eight duties
internal/jobs/retention_integration_test.go
internal/store/queries/retention.sql  purge/prune queries
internal/bench/bench_integration_test.go   //go:build integration && bench
internal/bench/seed.go                100k-row seeder (CopyFrom)
internal/app/logassert_integration_test.go  no-secret-leak sweep
internal/app/app.go                   (modify) scheduler start after listener, stop in drain
Makefile                              (modify) bench target
docs/trace-waivers.txt                (modify) shrink
```

---

### Task 1: Scheduler with panic-recovered tickers

**Files:**
- Create: `internal/jobs/scheduler.go`
- Test: `internal/jobs/scheduler_test.go`
- Modify: `internal/app/app.go`

**Interfaces:**
- Produces:
  - `jobs.Job interface{ Name() string; Run(ctx context.Context) }` — a Run is one tick's work; it logs its own counts.
  - `jobs.NewScheduler(log *slog.Logger) *Scheduler`; `(*Scheduler).Register(j Job, every time.Duration, initialDelay time.Duration)`; `(*Scheduler).Start(ctx)` — one goroutine per job: wait `initialDelay`, then tick `every`; each invocation wrapped: `defer recover → log error + stack, continue`; logs `job start`/`job finish` with `job`, `duration_ms`. `(*Scheduler).Stop()` — cancels the internal context and waits (WaitGroup) for in-flight ticks to return (EC-14 step 2: tickers stop before drain).
  - `app.Run` wiring: scheduler starts immediately after the listener opens; the drain path calls `scheduler.Stop()` between the health flip and `server.Shutdown` — replacing P1's seam comment.
- [ ] **Step 1: failing unit tests** — `TestBR_RUNTIME_5_TickerRunsAndRecovers`: a job that panics on tick 1 and increments a counter on tick 2 (10 ms cadence) reaches tick 2, and the error log carries the panic; `Stop()` returns only after an in-flight slow tick completes; post-Stop no further ticks.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: in-process job scheduler with panic-recovered ticks (BR-RUNTIME-5)"`

---

### Task 2: Retention duties 4–7 (expiry purges)

**Files:**
- Create: `internal/jobs/retention.go` (skeleton + duties 4–7), `internal/store/queries/retention.sql`
- Test: `internal/jobs/retention_integration_test.go`

**Interfaces:**
- sqlc (`retention.sql`): `PurgeExpiredSessions(idleBefore, absoluteBefore time.Time) (rows)`, `PurgeUsedOrExpiredResetTokens(now)`, `PurgeStaleRefreshTokens(before)` (rotated-or-revoked AND issued/rotated > 30 d), `PurgeStaleIdempotencyKeys(before)` — all `:execrows` so counts flow to telemetry.
- Produces: `jobs.NewRetention(pool, q *store.Queries, cache *schema.Cache, storage RetentionStorage, cfg RetentionConfig, log *slog.Logger) *Retention` implementing `jobs.Job` (`Name() == "retention"`); `RetentionConfig{TrashRetentionDays, RevisionLimit int}` (from `app.Config`); `RetentionStorage interface{ Delete(ctx, key string) error; Head(ctx, key string) (int64, string, error) }` — nil in media-less mode (duties 3/8 skip with one info line). `Run` executes duties in the documented order, each in its own error boundary (one duty failing logs and does not stop the rest), accumulating a `counts` map logged at finish.
- [ ] **Step 1: failing integration tests** — seed each table with one stale and one live row (sessions idle-expired vs fresh; reset used vs pending; refresh rotated-31d-ago vs live; idempotency 25 h vs 1 h) → one `Run` → stale gone, live intact (four `t.Run`s under `TestBR_LIFE_8_ExpiryPurges` — BR-AUTH-5/13 cross-references in subtest names).
- [ ] **Step 2–4:** implement (+`make generate`), PASS. **Step 5: Commit** — `git commit -m "feat: retention duties 4-7 — session/token/idempotency expiry purges"`

---

### Task 3: Retention duties 1–2 (trash purge, revision pruning)

**Files:**
- Modify: `internal/jobs/retention.go`, `internal/store/queries/retention.sql`
- Test: extend `retention_integration_test.go`

**Interfaces:**
- Duty 1 iterates the schema snapshot's collections; per collection, candidate rows come via a `query.Builder` trash-scope build (`deleted_at < cutoff`) — collection SQL stays in `internal/query` (BR-SCHEMA-3); each candidate is deleted individually (`lifecycle.Purge` semantics inline: hard delete + `DeleteRevisionsForRecord`); SQLSTATE 23503 → skip + `warn` log `{"msg":"retention skip","reason":"fk_restrict","collection":…,"record_id":…,"blocking_table":…}` — the alertable line (08 §Alerting item 3).
- Duty 2: sqlc `ListPrunableRevisions(collection_id, record_id, keep int)` is awkward across records — instead one set-based query per collection: `DELETE FROM cms_revisions WHERE id IN (SELECT id FROM (SELECT id, row_number() OVER (PARTITION BY collection_id, record_id ORDER BY version_no DESC) rn, published, first_value(version_no) OVER (...) FROM cms_revisions WHERE collection_id=$1) t WHERE rn > $2 AND NOT published)` — never the published one (predicate) and never the newest (rn=1 is newest; keep = `CMS_REVISION_LIMIT` so `rn > limit` spares the newest by construction). Exact SQL in `retention.sql` as `PruneRevisions :execrows`.
- [ ] **Step 1: failing integration tests** — `TestBR_LIFE_8_TrashPurgeHonorsFKRestrict`: two trashed-31-days records, one referenced by a relation field → one purged, one skipped with the warn line (assert via captured logger) and still present; `TestBR_LIFE_8_RevisionPruningExclusions`: record with 55 revisions, published at version 10 → after Run: 50 remain and both version 55 (newest) and version 10 (published) survive; oldest unpublished went first.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: retention duties 1-2 — FK-safe trash purge and revision pruning (BR-LIFE-8)"`

---

### Task 4: Retention duties 3 + 8 (media orphans, deletion-outbox retry)

**Files:**
- Modify: `internal/jobs/retention.go`
- Test: extend `retention_integration_test.go` (MinIO-gated: skip without `TEST_S3_ENDPOINT`)

**Interfaces:**
- Duty 3: `ListStalePendingMedia(before)` (add to `media.sql`) → per row: `storage.Delete(object_key)` (404-success) → delete row — object FIRST, then row (BR-MEDIA-2 order: a crash between leaves a re-sweepable row, never an orphaned object).
- Duty 8: `ListMediaDeletionsOlderThan(before)` (exists — P7) → per row: `storage.Delete` (404-success) → `DeleteMediaDeletion` (BR-MEDIA-5 retry half).
- Media-less (`storage == nil`): both duties log one `info` skip line.
- [ ] **Step 1: failing integration tests** — `TestBR_MEDIA_2_OrphanSweep`: pending row 25 h old with a real uploaded object → Run → object gone, row gone; pending 1 h old → untouched; `TestBR_MEDIA_5_OutboxRetry`: seed an outbox row 2 h old (object present) → Run → object gone, queue row gone; outbox row with NO object (404 path) → queue row gone.
- [ ] **Step 2–4:** implement, PASS. **Step 5: Commit** — `git commit -m "feat: retention duties 3+8 — orphan sweep and deletion-outbox retry (BR-MEDIA-2/5)"`

---

### Task 5: Log-assertion sweep, bench, wiring, waiver shrink

**Files:**
- Create: `internal/app/logassert_integration_test.go`, `internal/bench/seed.go`, `internal/bench/bench_integration_test.go`
- Modify: `Makefile` (bench target), `docs/trace-waivers.txt` (remove BR-LIFE-8, BR-RUNTIME-5)

**Interfaces / content:**
- **Log assertions:** boot in-process with a buffer logger; drive the full credential surface (setup, login, csrf, API-key create, end-user register/login/refresh, presign when S3 present); then assert over the whole buffer: no regex `cms_[A-Za-z0-9_\-]{30,}` (API-key/plaintext shape), no `eyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\.` (JWT), no `X-Amz-Signature`, no `cms_session=`; carve out the two allowed warn lines by their `msg` values (`setup token issued…`, recovery equivalent) — hard rule 6 as a test (`TestNoSecretsInLogs`).
- **Bench** (`//go:build integration && bench`): `seed.go` — create a collection (indexed `title` text, indexed `rank` number, `body` richText) via `schema.Engine`, then `CopyFrom` 100k rows directly into `c_<slug>` — seeding is the ONE sanctioned bypass of `Document.Set`, test-build-tagged and justified in a comment (BR-RBAC-5 governs production paths); publish-status distribution 80/20. Bench test: httptest server over the real router; warm 50 requests; measure 300 single-record GETs (random published ids) and 300 list GETs (`?sort=rank&limit=25`, random offsets ≤ 1000); compute p95s; log both; assert `p95(single) ≤ 50ms && p95(list) ≤ 120ms` unless `BENCH_ASSERT=0` (`TestN3_N4_ReferenceLatencies`).
- Makefile: `bench: web/dist/index.html` → `./scripts/testenv.sh go test -tags "integration bench" -run 'TestN3_N4' ./internal/bench/ -v`.
- Smoke: retention wired into the running binary (scheduler registered in `app.Run`) — the existing smoke test gains an assertion that a forced `Run` (exported test hook or 1-min initial delay observed) purges a seeded expired session.
- [ ] **Steps:** write failing tests → implement seeder/bench → `make test`, `make bench`, `make trace`, `make lint` all green (waived −2; verify `grep -c '^BR-' docs/trace-waivers.txt` returns 0 — the file is empty of entries) → commit — `git commit -m "feat: P8 complete — retention live, no-secret log sweep, N-3/N-4 bench gate"`.

## Plan Self-Review Notes

- After P8 the waiver file is EMPTY — `trace.sh` exempts structural and V2/V3-tagged rules (P1 plan), so every V1-binding BR is test-covered before the UI phase and P9's "waiver reaches empty" gate condition holds literally. Task 5's verification asserts the file contains zero `BR-` lines.
- Bench asserts through the full HTTP stack (middleware included) — stricter than a query-only benchmark and honest to N-3/N-4's "answers in".
- CopyFrom seeding bypasses Document.Set deliberately (100k validated writes would dominate bench wall-clock); tagged test-only.
- Duty-1 per-row deletes trade throughput for precise FK-skip logging — at single-tenant trash volumes this is the right trade; noted for re-evaluation if V2 grows volume.
