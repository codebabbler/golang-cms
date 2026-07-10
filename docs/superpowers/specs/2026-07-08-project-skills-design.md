# Project Skills & CLAUDE.md — Design

- **Version:** 1.0
- **Date:** 2026-07-08
- **Owner:** Miraj Aryal
- **Status:** Approved pending user review

## 1. Purpose

Complete the implementation-enablement layer for golang-cms. Six skills shipped with the documentation set (Batch 7): `schema-ddl-safety`, `query-builder-invariants`, `sqlc-workflow`, `api-conventions`, `svelte-admin-conventions`, `br-traceability-testing`. This design adds the three missing skills, a root `CLAUDE.md` that makes the whole layer discoverable to future sessions, and one small documentation addendum for a gap the design work exposed (first-admin bootstrap).

## 2. Decisions

- **D-1 Scope: gap skills + CLAUDE.md.** Skills nobody discovers don't fire; CLAUDE.md wires the layer into every session. The existing six skills are not reworked — revision waits until implementation exposes weaknesses.
- **D-2 Gap skills: exactly three.** `content-lifecycle-invariants`, `auth-security-conventions`, `run-and-verify`. A standalone integration-test-harness skill was rejected; its mechanics fold into `run-and-verify`, which owns the same disposable-Postgres lifecycle.
- **D-3 Structure: thin router.** CLAUDE.md stays ~60 lines and routes; depth lives in the nine skills and fourteen docs. Authority chain: `BUSINESS_RULES.md` > `docs/architecture/*` > skills > code comments. Rejected: fat CLAUDE.md (duplication/drift, per-session token tax) and per-directory CLAUDE.md files (directories don't exist yet; skill descriptions already route).
- **D-4 First-admin bootstrap (new system behavior).** When `cms_users` is empty at startup, the binary logs a one-time setup token; the admin UI's `/setup` screen consumes it to create the first super admin. Single-use, expires on use or restart, nothing persists, no new env vars. Rejected: bootstrap env vars (config surface + secret-in-env) and a CLI subcommand (command surface).
- **D-5 No new invariants in skills.** All three skills distill existing documents, consistent with the Batch 7 rule. The only new system behavior is D-4, which enters the *documents* first via the addendum, then the skills cite it.
- **D-6 Git handling.** All files land in the working tree uncommitted; the user manages git.

## 3. Skill Specifications

All three follow the established format: YAML frontmatter (`name`, trigger-phrased `description`), "distilled from" source citations, hard rules, working rules, test obligations, review checklist.

### 3.1 `content-lifecycle-invariants`

- **Scope:** writes and state transitions in `internal/lifecycle`. Read-side filtering belongs to `query-builder-invariants`; the skill states this boundary.
- **Sources:** `docs/architecture/07-data-model.md`, `docs/architecture/03-dynamic-schema.md`, `docs/BUSINESS_RULES.md` §3 (BR-LIFE-1…9).
- **Must encode:** the write transaction (live-row compare-and-set + revision insert, one transaction — BR-LIFE-1/7); publish semantics (revision data copy + status flip + `published`-flag move, atomic, at most one published revision per record); pending-draft detection (newest `version_no` > published `version_no`); the four restore drift-mapping rules with audit-detail recording (EC-5); trash/restore/purge (tombstone, 409 naming the colliding field, FK RESTRICT purge — EC-6); retention exclusions (never the published revision, never the newest); audit emission on every method (BR-AUDIT-1); V2 scheduled-publish catch-up (BR-LIFE-9).

### 3.2 `auth-security-conventions`

- **Scope:** auth internals — `internal/auth`, token handling, credential storage. Route wiring and middleware order belong to `api-conventions`; the skill states this boundary.
- **Sources:** `docs/architecture/05-auth-security.md`, `docs/BUSINESS_RULES.md` §4 (BR-AUTH-1…10), `docs/architecture/07-data-model.md` (token tables).
- **Must encode:** the hashing discipline table (Argon2id for passwords; sha256 for session, API-key, and refresh tokens; raw values never persist and never log); exact session-cookie attributes; CSRF-hash validation against `cms_sessions`; expiry windows (7 d idle / 30 d absolute / 4 h recent-auth); API-key show-once and revoked-rows-persist; JWT rules (RS256, 15-minute TTL, identity-only claims, `cms_system_keys` autogeneration); refresh rotation with family revocation on reuse (EC-8); the deterministic `X-Forwarded-For` trust rule (EC-10); the first-admin bootstrap flow (D-4, after the addendum lands).

### 3.3 `run-and-verify`

- **Scope:** the project skill the generic `/run` and `/verify` harness skills discover; description phrased as their trigger ("running, launching, smoke-testing, or verifying this app locally").
- **Sources:** `docs/architecture/09-deployment.md`, `docs/architecture/10-project-structure.md`, `docs/architecture/08-observability.md`, `docs/REQUIREMENTS.md` §6.
- **Must encode:** prerequisites (Go, Node, Docker); the disposable-PostgreSQL-16 container procedure serving both dev runs and integration tests; the minimal env set (`DATABASE_URL`, `CMS_SESSION_SECRET`; media flows documented as skipped without the S3 variables); `make build` / `make dev` / `make test` / `make trace`; the smoke sequence mapped to UAC-1.x: healthz → setup/login → create collection → record CRUD → publish → public fetch → trash/restore → optimistic-lock 409; the bootstrap token flow for first login (D-4).
- **Caveat clause:** the skill encodes the contract of docs 09/10 before code exists and must be validated and corrected during Phase 1 of implementation. The skill itself carries this caveat visibly.

## 4. CLAUDE.md Specification

Root `CLAUDE.md`, ~60 lines, exactly six sections:

1. **Identity** — three sentences: headless config-driven CMS, single Go binary, single tenant; PostgreSQL 16+ and S3-compatible storage as the only runtime dependencies; docs complete, implementation pending.
2. **Authority chain** — `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code comments; on conflict the higher source wins and the lower one gets fixed.
3. **Document map** — the 14 documents, one line each, kept consistent with `docs/architecture/00-README.md`.
4. **Skill index** — all nine skills with firing conditions.
5. **Always-on hard rules** — six one-liners: collection-table SQL only via `query.Builder`; writes only via `Document.Set`; DDL only via `schema.Engine`'s whitelist; errors only via `httpapi.WriteError`; no runtime dependencies beyond PostgreSQL + S3; never log tokens, secrets, or presigned URLs.
6. **Workflow & commands** — `make build/test/trace/generate/dev`; git convention: the user commits, sessions never commit unasked; spec-driven flow (brainstorm → spec → plan → subagent-driven execution for code); `make trace` gates completion.

Exclusions (deliberate): rule detail that lives in skills/docs, PRD content, batch-execution mechanics.

## 5. Documentation Addendum (first-admin bootstrap)

Four small edits, entering the documents before the skills cite the behavior:

1. `docs/BUSINESS_RULES.md` — new rule **BR-AUTH-11**: "When `cms_users` is empty at startup, the system logs a single-use setup token and enables `/setup`; consuming it creates the first super admin and disables the route. The token dies on use or process exit." *Enforcement:* `auth.Bootstrap` at startup + the `/setup` handler.
2. `docs/architecture/05-auth-security.md` — a "First-Admin Bootstrap" paragraph in §1 specifying: token is 256-bit random, logged once at `warn`, held in memory only; `/setup` accepts it exactly once; the route returns 404 whenever `cms_users` is non-empty.
3. `docs/architecture/09-deployment.md` — startup list gains the bootstrap check between schema-cache load and catch-up scan.
4. `docs/architecture/06-admin-ui.md` — screens table gains the `/setup` row (token + first-admin form, shown only on fresh systems).

## 6. Acceptance Criteria

1. Three new skill directories exist under `.claude/skills/`, each following the established format with trigger-phrased descriptions; `run-and-verify` carries the Phase-1 validation caveat.
2. None of the three new skills introduces an invariant absent from the docs (D-5); every hard rule in a skill cites a BR identifier or doc section.
3. `CLAUDE.md` exists at the repo root with exactly the six sections of §4, ≤ 70 lines, and no rule detail duplicated from skills beyond the six one-liners.
4. The four addendum edits of §5 are applied; BR-AUTH-11 appears under BUSINESS_RULES §4 Authentication (Naming Constants untouched — no new env vars).
5. Scope boundaries are stated inside both `content-lifecycle-invariants` (vs. query-builder) and `auth-security-conventions` (vs. api-conventions).
6. Nothing is committed to git; all changes land in the working tree only.

## 7. Out of Scope

- Reworking the six existing skills.
- Implementation of the bootstrap flow (lands with V1 Phase 1 code).
- Per-directory CLAUDE.md files.
