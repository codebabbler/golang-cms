# Project Skills & CLAUDE.md Enablement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver the three gap skills, root CLAUDE.md, and first-admin-bootstrap doc addendum per `docs/superpowers/specs/2026-07-08-project-skills-design.md`.

**Architecture:** Six ordered tasks. Task 1 applies the four-edit documentation addendum (BR-AUTH-11) so later skills cite existing rules (spec D-5). Tasks 2–4 write the three skills with exact content. Task 5 writes CLAUDE.md. Task 6 runs the acceptance verification.

**Tech Stack:** Markdown only. Verification via `grep`, `wc`, `diff`.

## Global Constraints

- **No git operations.** Nothing is committed or staged in any task; the user manages git (spec D-6, acceptance criterion 6).
- Skills follow the established format: YAML frontmatter (`name`, trigger-phrased `description`), "Distilled from" line naming authoritative docs, hard/working rules, test obligations where applicable, review checklist.
- No skill introduces an invariant absent from the docs (spec D-5); every hard rule cites a BR identifier or doc section.
- CLAUDE.md: exactly six sections, ≤ 70 lines total.
- No new env vars anywhere (spec D-4).
- Exact strings from the spec: rule ID `BR-AUTH-11`; skill names `content-lifecycle-invariants`, `auth-security-conventions`, `run-and-verify`.

---

### Task 1: Documentation addendum — first-admin bootstrap (BR-AUTH-11)

**Files:**
- Modify: `docs/BUSINESS_RULES.md` (§4 Authentication, after BR-AUTH-10)
- Modify: `docs/architecture/05-auth-security.md` (end of §1 Admin Sessions)
- Modify: `docs/architecture/09-deployment.md` (Startup list)
- Modify: `docs/architecture/06-admin-ui.md` (Screens table)

**Interfaces:**
- Consumes: nothing.
- Produces: rule ID `BR-AUTH-11` present in four documents; Tasks 3 and 4 cite it.

- [ ] **Step 1: Add BR-AUTH-11 to BUSINESS_RULES.md**

Find this block (end of §4):

```markdown
- **BR-AUTH-10.** The RSA-2048 keypair persists in `cms_system_keys` (the system-table form of the auth specification's `system_keys`); the system auto-generates it when `JWT_PRIVATE_KEY` is absent.
  *Enforcement:* `auth.Keys.Load` at startup.
```

Append directly after it:

```markdown
- **BR-AUTH-11.** When `cms_users` is empty at startup, the system logs a single-use setup token and enables `/setup`; consuming it creates the first super admin and disables the route. The token dies on use or process exit.
  *Enforcement:* `auth.Bootstrap` at startup + the `/setup` handler.
```

- [ ] **Step 2: Add the bootstrap paragraph to 05-auth-security.md**

Find the last line of §1 (`- **Rate limiting:** 10 attempts/15 min per email, 30 attempts/15 min per IP (BR-AUTH-6), evaluated before Argon2id work (\`04-api-layer.md\` middleware order).`) and append after it:

```markdown
- **First-Admin Bootstrap (BR-AUTH-11):** when `cms_users` is empty at startup, `auth.Bootstrap` generates a 256-bit random setup token, logs it exactly once at `warn`, and holds it in memory only — nothing persists. The admin UI's `/setup` screen accepts it exactly once to create the first super admin; the route returns 404 whenever `cms_users` is non-empty, and the token dies on use or process exit.
```

- [ ] **Step 3: Insert the bootstrap step into 09-deployment.md startup list**

Replace:

```markdown
3. Load the schema cache.
4. Run the scheduled-publish catch-up scan: publish every record whose `publish_at` elapsed while the binary was down, logging each late publication (BR-LIFE-9; `08-observability.md`). *(Resolves EC-13, deployment half.)*
5. Open the HTTP listener on `CMS_PORT`; `/healthz` begins returning 200.
```

with:

```markdown
3. Load the schema cache.
4. When `cms_users` is empty, generate and log the single-use setup token and enable `/setup` (BR-AUTH-11).
5. Run the scheduled-publish catch-up scan: publish every record whose `publish_at` elapsed while the binary was down, logging each late publication (BR-LIFE-9; `08-observability.md`). *(Resolves EC-13, deployment half.)*
6. Open the HTTP listener on `CMS_PORT`; `/healthz` begins returning 200.
```

- [ ] **Step 4: Add the Setup row to 06-admin-ui.md screens table**

Find the `| Access rules |` row and append this row directly after it:

```markdown
| Setup (`/setup`) | Shown only on fresh systems (`cms_users` empty): accepts the logged single-use setup token and creates the first super admin (BR-AUTH-11). Returns 404 whenever `cms_users` is non-empty. |
```

- [ ] **Step 5: Verify**

Run: `grep -l 'BR-AUTH-11' docs/BUSINESS_RULES.md docs/architecture/05-auth-security.md docs/architecture/09-deployment.md docs/architecture/06-admin-ui.md | wc -l`
Expected: `4`

Run: `awk '/^## Startup/,/^## Shutdown/' docs/architecture/09-deployment.md | grep -c '^[0-9]\.'`
Expected: `6` (startup list now has six numbered items; the awk scope excludes the reverse-proxy contract's own numbered list)

Run: `grep -rn 'CMS_BOOTSTRAP\|SETUP_TOKEN' docs/BUSINESS_RULES.md docs/REQUIREMENTS.md docs/architecture/`
Expected: no output — no new env vars in product docs (spec D-4; the plan file itself contains this pattern and is excluded)

---

### Task 2: `content-lifecycle-invariants` skill

**Files:**
- Create: `.claude/skills/content-lifecycle-invariants/SKILL.md`

**Interfaces:**
- Consumes: BR-LIFE-1…9, BR-AUDIT-1 (existing rules; Task 1 not required).
- Produces: skill name `content-lifecycle-invariants` referenced by Task 5's skill index.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
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
````

- [ ] **Step 2: Verify**

Run: `head -4 .claude/skills/content-lifecycle-invariants/SKILL.md | grep '^name: content-lifecycle-invariants'`
Expected: one matching line

Run: `grep -c 'BR-LIFE-\|BR-AUDIT-' .claude/skills/content-lifecycle-invariants/SKILL.md`
Expected: ≥ 8 (every hard rule cites)

Run: `grep -n 'query-builder-invariants' .claude/skills/content-lifecycle-invariants/SKILL.md`
Expected: the boundary statement line (acceptance criterion 5)

---

### Task 3: `auth-security-conventions` skill

**Files:**
- Create: `.claude/skills/auth-security-conventions/SKILL.md`

**Interfaces:**
- Consumes: BR-AUTH-1…11 (Task 1 must be complete — BR-AUTH-11 must exist before this cites it).
- Produces: skill name `auth-security-conventions` referenced by Task 5's skill index.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
---
name: auth-security-conventions
description: Use when touching internal/auth or any credential/token handling — password hashing, session issuance, API keys, JWTs, refresh rotation, or the first-admin bootstrap. Encodes the hashing discipline, token-family revocation, and key management rules.
---

# Auth Security Conventions

Distilled from `docs/architecture/05-auth-security.md`, `docs/BUSINESS_RULES.md` §4 (BR-AUTH-1…11), and `docs/architecture/07-data-model.md` (token tables). Those documents are authoritative.

**Boundary:** this skill owns auth *internals* — issuance, verification, storage. Route wiring and the normative middleware order belong to `api-conventions` — do not restate them here.

## The Hashing Discipline (memorize this table)

| Credential | At rest | Where |
|---|---|---|
| Admin/end-user passwords | Argon2id, per-hash salts (BR-AUTH-3) | `auth.Password` — the only hashing path |
| Session tokens | sha256 (BR-AUTH-2) | `cms_sessions.token_hash` |
| API keys | sha256 (BR-AUTH-7) | `cms_api_keys.token_hash` |
| Refresh tokens | sha256 (BR-AUTH-9) | `cms_refresh_tokens.token_hash` |

Raw token values never persist and never log — no exceptions, any level (`docs/architecture/08-observability.md`). `CMS_SESSION_SECRET` signs cookies; it plays no role in password hashing.

## Hard Rules

1. **Session cookie attributes are exact** (BR-AUTH-1): `cms_session=<256-bit-random>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`, body carries `csrfToken`. CSRF validates against `cms_sessions.csrf_hash` (BR-AUTH-4).
2. **Expiry windows** (BR-AUTH-5): 7 days idle, 30 days absolute, 4-hour recent-auth for destructive operations. These constants are business rules, not tunables.
3. **API keys** (BR-AUTH-7): plaintext displays exactly once at creation; the `cms_` prefix stays (leak-greppability); revocation sets `revoked_at` and the row persists for audit — never delete key rows.
4. **JWTs carry identity, never permissions** (BR-AUTH-8): RS256, 15-minute TTL; `access.Evaluator` resolves permissions per request. Adding a role or scope claim to the JWT is a defect even if it "saves a query."
5. **Key management** (BR-AUTH-10): the RSA-2048 keypair lives in `cms_system_keys`, auto-generated when `JWT_PRIVATE_KEY` is absent. RS256 stands until a concrete constraint justifies EdDSA.
6. **Refresh rotation + family revocation** (BR-AUTH-9, EC-8): every refresh rotates both tokens; presenting an already-rotated token revokes the entire `family_id` and returns 401. The family check precedes issuance — order matters.
7. **First-admin bootstrap** (BR-AUTH-11): when `cms_users` is empty at startup, `auth.Bootstrap` logs a 256-bit single-use setup token (once, at `warn`, memory-only); `/setup` consumes it to create the first super admin and returns 404 whenever `cms_users` is non-empty. The token dies on use or process exit. No bootstrap env vars exist — do not add any.
8. **Client IP resolution** (EC-10): private/loopback direct peer → rightmost non-private `X-Forwarded-For` entry; public direct peer → socket address, header ignored. Deterministic, no trusted-proxy configuration.

## Test Obligations

Trace to BR-AUTH-1…11. Highest-value adversarial tests: replaying a rotated refresh token kills the family; a database scan asserting no raw token substrings; `/setup` returning 404 with a non-empty `cms_users` even with a valid token; spoofed `X-Forwarded-For` from a public peer not moving the rate-limit key.

## Review Checklist

- [ ] Every stored credential appears in the hashing table with the right algorithm?
- [ ] No raw token in any persistence or log path?
- [ ] JWT claims still identity-only?
- [ ] Family check before refresh issuance?
- [ ] Bootstrap route dead on non-empty `cms_users`?
- [ ] `make trace` green for touched BR-AUTH rules?
````

- [ ] **Step 2: Verify**

Run: `grep -c 'BR-AUTH-11' .claude/skills/auth-security-conventions/SKILL.md`
Expected: ≥ 1

Run: `grep -n 'api-conventions' .claude/skills/auth-security-conventions/SKILL.md`
Expected: the boundary statement line (acceptance criterion 5)

Run: `grep -o 'BR-AUTH-[0-9]*' .claude/skills/auth-security-conventions/SKILL.md | sort -uV | wc -l`
Expected: ≥ 9

---

### Task 4: `run-and-verify` skill

**Files:**
- Create: `.claude/skills/run-and-verify/SKILL.md`

**Interfaces:**
- Consumes: BR-AUTH-11 (Task 1), `make` targets from `docs/architecture/10-project-structure.md`.
- Produces: skill name `run-and-verify` referenced by Task 5's skill index; the project skill `/run` and `/verify` discover.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
---
name: run-and-verify
description: Use when running, launching, smoke-testing, or verifying this app locally — starting the server, spinning up the database, or confirming a change works end-to-end.
---

# Run and Verify

Distilled from `docs/architecture/09-deployment.md`, `docs/architecture/10-project-structure.md`, `docs/architecture/08-observability.md`, and `docs/REQUIREMENTS.md` §6.

> **Phase-1 caveat:** this skill encodes the contract of the docs *before implementation exists*. During V1 Phase 1, validate every command against reality and update this file where the implementation legitimately diverged — the docs win over this skill on conflict.

## Prerequisites

Go toolchain matching `go.mod`, Node LTS (admin UI build), Docker (disposable PostgreSQL).

## Database (dev runs and integration tests share this)

```bash
docker run --rm -d --name cms-pg \
  -e POSTGRES_PASSWORD=cms -e POSTGRES_DB=cms \
  -p 5432:5432 postgres:16
export DATABASE_URL='postgres://postgres:cms@localhost:5432/cms?sslmode=disable'
```

Tear down with `docker stop cms-pg` (container is `--rm`). Never point a dev binary at a non-disposable database.

## Environment

Minimum to boot: `DATABASE_URL` plus `CMS_SESSION_SECRET` (any 32+ random bytes). All variables: `docs/BUSINESS_RULES.md` § Naming Constants. Without the `S3_*`/`R2_*` variables the binary boots and everything except media flows works — media smoke steps below are skipped in that case.

## Build and Run

```bash
make build        # vite build → go build with embeds; fails if web/dist missing
./bin/cms         # startup order per 09-deployment.md; /healthz → 200 when ready
make dev          # UI iteration: vite dev server proxying /api to a running binary
```

Startup failure exits non-zero listing every missing env var at once. Migration or key-load failure aborts startup — that is fail-closed behavior (N-11), not a bug.

## Smoke Sequence (maps to UAC-1.x)

1. `curl -fsS localhost:8080/healthz` → 200.
2. **Fresh system:** grab the single-use setup token from the startup `warn` log (BR-AUTH-11), open `/setup`, create the first super admin. Re-running against a used system: `/setup` is 404 — expected.
3. Log in; create a collection with a `text` field (UAC-1.1: table `c_<slug>` exists — verify with `psql "$DATABASE_URL" -c '\d c_<slug>'`).
4. Create a record → publish → fetch through `/api/collections/<slug>/records` unauthenticated → the published record returns (UAC-1.2).
5. Update with a stale `version` → expect `409 conflict` (UAC-1.4).
6. Trash the record → gone from public list; restore → back (UAC-1.4).
7. With `S3_*` configured: presign → PUT → finalize → attach to a `media` field (UAC-1.5). Skip without S3 env.

## Code-Level Verification

```bash
make test    # unit + integration (//go:build integration) against the container above
make trace   # BR coverage gate — done means green
```

A change is verified when the smoke sequence relevant to it passes AND `make test && make trace` are green — not when it compiles.
````

- [ ] **Step 2: Verify**

Run: `grep -n 'Phase-1 caveat' .claude/skills/run-and-verify/SKILL.md`
Expected: the caveat block line (acceptance criterion 1)

Run: `grep -c 'UAC-1\.' .claude/skills/run-and-verify/SKILL.md`
Expected: ≥ 4

Run: `grep -n 'BR-AUTH-11' .claude/skills/run-and-verify/SKILL.md`
Expected: ≥ 1 match (bootstrap step cites the rule)

---

### Task 5: Root CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

**Interfaces:**
- Consumes: all nine skill names (Tasks 2–4 complete), doc filenames.
- Produces: the always-loaded router; Task 6 verifies its bounds.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
# golang-cms

Headless, config-driven CMS: one Go binary, one tenant. Admins define collections at runtime; the schema engine provisions real Postgres tables. PostgreSQL 16+ and S3-compatible storage are the only runtime dependencies. Status: documentation complete (`docs/`), implementation not started.

## Authority Chain

`docs/BUSINESS_RULES.md` > `docs/architecture/*` > `.claude/skills/*` > code comments. On conflict, the higher source wins and the lower one gets fixed.

## Documentation Map

| Doc | Contents |
|---|---|
| `docs/BUSINESS_RULES.md` | All invariants (BR-IDs) with enforcement points; tests trace to these |
| `docs/REQUIREMENTS.md` | PRD: personas, F/N/O requirements, acceptance criteria (UAC) |
| `docs/architecture/00-README.md` | Reading order and identifier conventions |
| `docs/architecture/01-system-overview.md` | Architecture, request flows, Edge-Case Register (EC-1…15) |
| `docs/architecture/02-core-interfaces.md` | QueryBuilder, Decision evaluator, Document.Set, service contracts |
| `docs/architecture/03-dynamic-schema.md` | Runtime-DDL engine: whitelist, conversion matrix, drift mapping |
| `docs/architecture/04-api-layer.md` | Routes, middleware order, envelope, pagination, uploads |
| `docs/architecture/05-auth-security.md` | Sessions, API keys, JWTs, RBAC, bootstrap, threat model |
| `docs/architecture/06-admin-ui.md` | Svelte 5 SPA conventions and screens |
| `docs/architecture/07-data-model.md` | System tables, collection anatomy, live-table/revisions contract |
| `docs/architecture/08-observability.md` | slog schema, audit stream, job telemetry |
| `docs/architecture/09-deployment.md` | Build, startup/shutdown, proxy contract, cache busting |
| `docs/architecture/10-project-structure.md` | Package layout, import rules, Makefile |
| `docs/architecture/11-roadmap.md` | V1/V2/V3 scopes and delivery gates |

## Skills (invoke before touching the subsystem)

| Skill | Fires when |
|---|---|
| `schema-ddl-safety` | `internal/schema` — DDL engine, slugs, type changes |
| `query-builder-invariants` | `internal/query` or any collection list/read path |
| `content-lifecycle-invariants` | `internal/lifecycle` — saves, publish, trash, restore |
| `auth-security-conventions` | `internal/auth` — tokens, hashing, credentials, bootstrap |
| `api-conventions` | `internal/httpapi` — handlers, routes, middleware |
| `sqlc-workflow` | `cms_*` system tables, migrations, `internal/store` |
| `svelte-admin-conventions` | `web/` — admin UI |
| `br-traceability-testing` | writing tests or changing BUSINESS_RULES.md |
| `run-and-verify` | running, smoke-testing, or verifying the app locally |

## Hard Rules (always on)

1. SQL for collection tables exists only inside `internal/query` — BR-SCHEMA-3.
2. Every content write goes through `content.Document.Set` — BR-RBAC-5.
3. DDL runs only through `schema.Engine`'s closed whitelist — BR-SCHEMA-4.
4. Error responses only via `httpapi.WriteError` with the closed code registry — BR-API-3.
5. No runtime dependencies beyond PostgreSQL 16+ and S3-compatible storage — BR-RUNTIME-2.
6. Never log tokens, cookie values, presigned URLs, or JWT bodies.

## Workflow

- Commands: `make build` (vite→go), `make test` (unit+integration, disposable PG16), `make trace` (BR coverage gate), `make generate` (sqlc), `make dev`.
- Git: the user commits; never commit or stage unless explicitly asked.
- Flow: brainstorm → spec (`docs/superpowers/specs/`) → plan (`docs/superpowers/plans/`) → subagent-driven execution for code.
- Done = smoke-relevant checks pass AND `make test && make trace` green — not "it compiles."
````

- [ ] **Step 2: Verify**

Run: `wc -l < CLAUDE.md`
Expected: ≤ 70

Run: `grep -c '^## ' CLAUDE.md`
Expected: `5` (Authority Chain, Documentation Map, Skills, Hard Rules, Workflow — the identity section is the title block)

Run: `grep -c '| \`' CLAUDE.md`
Expected: `23` (14 doc rows + 9 skill rows)

---

### Task 6: Acceptance verification against the spec

**Files:**
- Verify only; no writes.

**Interfaces:**
- Consumes: all artifacts from Tasks 1–5.
- Produces: confirmation of spec §6 criteria 1–6.

- [ ] **Step 1: Criterion 1 — three skills exist, correct names, caveat present**

Run: `ls .claude/skills/ | sort`
Expected: 9 directories including `content-lifecycle-invariants`, `auth-security-conventions`, `run-and-verify`

Run: `grep -l 'Phase-1 caveat' .claude/skills/run-and-verify/SKILL.md`
Expected: the file path

- [ ] **Step 2: Criterion 2 — no uncited hard rules**

Run: `grep -c 'BR-\|EC-\|UAC-\|docs/' .claude/skills/content-lifecycle-invariants/SKILL.md .claude/skills/auth-security-conventions/SKILL.md .claude/skills/run-and-verify/SKILL.md`
Expected: every file ≥ 8 citation lines (run-and-verify cites via UAC-1.x and doc paths, not only BR/EC identifiers)

- [ ] **Step 3: Criterion 3 — CLAUDE.md bounds**

Run: `wc -l < CLAUDE.md`
Expected: ≤ 70

- [ ] **Step 4: Criterion 4 — addendum applied, no new env vars**

Run: `grep -l 'BR-AUTH-11' docs/BUSINESS_RULES.md docs/architecture/05-auth-security.md docs/architecture/09-deployment.md docs/architecture/06-admin-ui.md | wc -l`
Expected: `4`

Run: `grep -c '| \`CMS_' docs/BUSINESS_RULES.md`
Expected: `5` (CMS_SESSION_SECRET, CMS_PORT, CMS_LOG_LEVEL, CMS_TRASH_RETENTION_DAYS, CMS_REVISION_LIMIT — unchanged)

- [ ] **Step 5: Criterion 5 — boundary statements present**

Run: `grep -l 'Boundary:' .claude/skills/content-lifecycle-invariants/SKILL.md .claude/skills/auth-security-conventions/SKILL.md | wc -l`
Expected: `2`

- [ ] **Step 6: Criterion 6 — nothing staged or committed**

Run: `git status --short | grep -c '^[MADRC]'`
Expected: `0` (no staged entries; untracked `??` lines are fine)

Run: `git log --oneline -1`
Expected: `0a3ac5d Initial commit` (unchanged)
