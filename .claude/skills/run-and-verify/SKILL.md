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
