# P1 Foundation (Walking Skeleton) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One Go binary that boots per BR-RUNTIME-3 against PostgreSQL 16 (config → migrations → instance lock → listener), serves `/healthz`, `/readyz`, and the embedded Svelte shell under the EC-15 cache contract, drains per EC-14 — with `make build/test/trace/generate/dev` green.

**Architecture:** Greenfield implementation of Phase 1 of `docs/superpowers/specs/2026-07-11-v1-implementation-phasing-design.md`. Packages per `docs/architecture/10-project-structure.md`; startup/shutdown per `docs/architecture/09-deployment.md`; envelope/errors per `docs/architecture/04-api-layer.md`; DDL per `docs/architecture/07-data-model.md`; audit shape per `docs/architecture/02-core-interfaces.md` + `08-observability.md`.

**Tech Stack:** Go 1.25, chi v5, pgx v5, google/uuid, sqlc (go tool), Vite 7 + Svelte 5 (no SvelteKit — O-5), golangci-lint + depguard, disposable `postgres:16` via Docker for integration tests.

## Global Constraints

- Module path: `github.com/codebabbler/golang-cms` (from the git remote). Branch: `main` (project convention, user-authorized).
- Runtime services: PostgreSQL 16+ only in P1 (BR-RUNTIME-2). Go deps limited to chi v5, pgx v5, google/uuid. No squirrel (P4), no storage SDK (P7), no imaging libraries ever (BR-MEDIA-4).
- Advisory-lock keys (09 §Startup): migration `0x636D7300`, instance `0x636D7302`. (`0x636D7301` is P3's schema key — do not use.)
- Error registry (04): exactly nine codes — `validation_failed` 422, `unauthorized` 401, `forbidden` 403, `not_found` 404, `conflict` 409, `rate_limited` 429, `payload_too_large` 413, `internal` 500, `unavailable` 503. Written only by `httpapi.WriteError` (BR-API-3).
- Roles (BR-RBAC-1): exactly `super_admin`, `admin`, `editor`, `contributor`, `viewer`.
- Timeouts (09 §Timeouts, compiled-in): ReadHeader 5 s, Read 30 s, Write 30 s, Idle 120 s, per-request deadline 25 s, pgx pool max 10, drain window 15 s, instance-lock retry window 120 s. Default body cap 64 KiB.
- Cache contract (EC-15): `/assets/*` → `Cache-Control: public, max-age=31536000, immutable`; `index.html` → `Cache-Control: no-cache`. Identity encoding only — the edge owns compression.
- Env vars: the seventeen-variable table in `BUSINESS_RULES.md` §Naming Constants, verbatim names. Hard-required: `DATABASE_URL`, `CMS_MASTER_SECRET`. `S3_*` (four vars: `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`) optional as a unit; partial config fails startup listing the missing members (BR-MEDIA-6).
- Test naming: `TestBR_XXX_N_Description` or `t.Run("BR-XXX-N ...")` (10 §Testing Layout). Integration tests carry `//go:build integration`.
- Never log secrets (CLAUDE.md hard rule 6). No tokens exist yet in P1 — keep it that way.
- Commits: plain messages, no Co-Authored-By trailer. Never stage `.claude/skills/system-design/` or `prompt.md`.

## File Structure

```
Makefile                                   build/test/trace/generate/dev/lint
sqlc.yaml                                  sqlc v2 config → internal/store
.golangci.yml                              depguard seam rules
.gitignore                                 bin/, web/dist/, web/node_modules/
scripts/testdb.sh                          disposable postgres:16 wrapper
scripts/trace.sh                           BR coverage gate (D-P1-2)
docs/trace-waivers.txt                     BR-IDs pending later phases
cmd/cms/main.go                            env → slog → signal ctx → app.Run → exit code
internal/access/principal.go               Principal, Kind, Role constants (02 §Principal)
internal/app/config.go                     Config, LoadConfig, ConfigError
internal/app/app.go                        Run: startup steps 1/2/3/(4)/(5)/6, drain
internal/app/instancelock.go               acquire + watchdog (BR-RUNTIME-8)
internal/audit/audit.go                    Event, Sink, Recorder, slog sink, request-id ctx
internal/httpapi/errors.go                 ErrorCode registry + WriteError
internal/httpapi/envelope.go               WriteData envelope
internal/httpapi/middleware.go             RequestID, Logger, Recover, deadline, body cap
internal/httpapi/health.go                 HealthState, healthz/readyz handlers
internal/httpapi/spa.go                    SPA handler with EC-15 headers
internal/httpapi/router.go                 chi assembly in the 04 middleware order
internal/store/migrate.go                  embedded runner + cms_migrations ledger
internal/store/migrations/0001_system_tables.sql   13 tables per 07
internal/store/queries/users.sql           CountUsers (sqlc source)
internal/testdb/testdb.go                  per-test database helper
web/{package.json,vite.config.js,index.html,src/main.js,src/App.svelte}
web/embed.go                               //go:embed all:dist
```

---

### Task 1: Repo scaffold + web shell

**Files:**
- Create: `.gitignore`, `web/package.json`, `web/vite.config.js`, `web/index.html`, `web/src/main.js`, `web/src/App.svelte`, `Makefile`

**Interfaces:**
- Consumes: nothing (first task).
- Produces: `web/dist/` build output with content-hashed assets under `dist/assets/`; Makefile targets `web/dist/index.html` (build dep) and `dev`.

- [ ] **Step 1: Write `.gitignore`**

```gitignore
bin/
web/dist/
web/node_modules/
.superpowers/
```

- [ ] **Step 2: Write the Vite + Svelte 5 shell**

`web/package.json`:

```json
{
  "name": "golang-cms-admin",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "devDependencies": {
    "@sveltejs/vite-plugin-svelte": "^6.0.0",
    "svelte": "^5.0.0",
    "vite": "^7.0.0"
  }
}
```

`web/vite.config.js`:

```js
import { defineConfig } from 'vite'
import { svelte } from '@sveltejs/vite-plugin-svelte'

// make dev: vite serves the SPA and proxies /api to a locally running binary
// (10-project-structure.md §Makefile Targets).
export default defineConfig({
  plugins: [svelte()],
  server: {
    proxy: { '/api': 'http://localhost:8080' },
  },
})
```

`web/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>golang-cms admin</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.js"></script>
  </body>
</html>
```

`web/src/main.js`:

```js
import { mount } from 'svelte'
import App from './App.svelte'

mount(App, { target: document.getElementById('app') })
```

`web/src/App.svelte`:

```svelte
<main>
  <h1>golang-cms</h1>
  <p>Admin UI arrives in Phase 9. This shell proves the build pipeline.</p>
</main>
```

- [ ] **Step 3: Write the initial Makefile**

```make
.PHONY: build test unit trace generate dev lint

web/node_modules: web/package.json
	cd web && npm install
	touch web/node_modules

web/dist/index.html: web/node_modules web/index.html web/vite.config.js $(shell find web/src -type f)
	cd web && npm run build

dev:
	cd web && npm run dev
```

- [ ] **Step 4: Verify the shell builds with hashed assets**

Run: `make web/dist/index.html && ls web/dist/assets/`
Expected: build succeeds; `ls` shows files like `index-D2fA9k3x.js` (content-hashed names). `web/dist/index.html` references them.

Run: `git status --short`
Expected: `web/dist/` and `web/node_modules/` do NOT appear (gitignored).

- [ ] **Step 5: Commit**

```bash
git add .gitignore Makefile web/package.json web/vite.config.js web/index.html web/src/ web/package-lock.json
git commit -m "feat: repo scaffold and Vite/Svelte 5 admin shell"
```

---

### Task 2: Go module, access.Principal, config loader

**Files:**
- Create: `go.mod` (via `go mod init`), `internal/access/principal.go`, `internal/app/config.go`
- Test: `internal/app/config_test.go`

**Interfaces:**
- Consumes: nothing.
- Produces:
  - `access.Kind` (string type) with constants `KindAdmin="admin"`, `KindAPIKey="api_key"`, `KindEndUser="end_user"`, `KindAnonymous="anonymous"`; `access.Role` with `RoleSuperAdmin="super_admin"`, `RoleAdmin="admin"`, `RoleEditor="editor"`, `RoleContributor="contributor"`, `RoleViewer="viewer"`; `access.Principal{Kind Kind; ID uuid.UUID; Role Role}` (Scopes arrives in P5 per 02 §Principal).
  - `app.LoadConfig(lookup func(string) (string, bool)) (*Config, error)` — error is `*ConfigError` when env is invalid.
  - `app.Config{DatabaseURL, MasterSecret, JWTPrivateKey, JWTPublicKey string; S3 *S3Config; R2AccountID, R2PublicBucketURL, RecoveryEmail string; TrustedProxyCIDRs []*net.IPNet; EndUserRegistration bool; Port int; LogLevel slog.Level; TrashRetentionDays, RevisionLimit int}`; `S3Config{Endpoint, Bucket, AccessKey, SecretKey string}`; `(*Config).MediaEnabled() bool` (S3 != nil).
  - `app.ConfigError{Problems []string}` implementing `error` — one problem line per missing/invalid variable.

- [ ] **Step 1: Init the module and get deps**

```bash
go mod init github.com/codebabbler/golang-cms
go get github.com/google/uuid@latest github.com/jackc/pgx/v5@latest github.com/go-chi/chi/v5@latest
```

- [ ] **Step 2: Write `internal/access/principal.go`**

```go
// Package access holds the principal model and role constants (02-core-interfaces.md
// §Principal). The Evaluator and Decision types arrive in Phase 6.
package access

import "github.com/google/uuid"

type Kind string

const (
	KindAdmin     Kind = "admin"
	KindAPIKey    Kind = "api_key"
	KindEndUser   Kind = "end_user"
	KindAnonymous Kind = "anonymous"
)

// Role — exactly five roles exist (BR-RBAC-1).
type Role string

const (
	RoleSuperAdmin  Role = "super_admin"
	RoleAdmin       Role = "admin"
	RoleEditor      Role = "editor"
	RoleContributor Role = "contributor"
	RoleViewer      Role = "viewer"
)

// Principal is the resolved caller identity threaded through context.
// Scopes ([]CollectionScope, api_key kind only) arrives in Phase 5.
type Principal struct {
	Kind Kind
	ID   uuid.UUID // zero for anonymous
	Role Role      // admin kind only
}
```

- [ ] **Step 3: Write the failing config tests**

`internal/app/config_test.go`:

```go
package app

import (
	"strings"
	"testing"
)

func env(m map[string]string) func(string) (string, bool) {
	return func(k string) (string, bool) { v, ok := m[k]; return v, ok }
}

var minimalEnv = map[string]string{
	"DATABASE_URL":      "postgres://localhost/cms",
	"CMS_MASTER_SECRET": "0123456789abcdef0123456789abcdef",
}

func TestLoadConfig_MinimalEnvAppliesDefaults(t *testing.T) {
	cfg, err := LoadConfig(env(minimalEnv))
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if cfg.Port != 8080 {
		t.Errorf("Port = %d, want 8080", cfg.Port)
	}
	if cfg.TrashRetentionDays != 30 {
		t.Errorf("TrashRetentionDays = %d, want 30", cfg.TrashRetentionDays)
	}
	if cfg.RevisionLimit != 50 {
		t.Errorf("RevisionLimit = %d, want 50", cfg.RevisionLimit)
	}
	if cfg.EndUserRegistration {
		t.Error("EndUserRegistration should default to disabled (BR-AUTH-14)")
	}
	if cfg.MediaEnabled() {
		t.Error("MediaEnabled must be false with no S3_* vars (BR-MEDIA-6)")
	}
}

func TestLoadConfig_MissingRequiredListsEveryProblem(t *testing.T) {
	_, err := LoadConfig(env(map[string]string{}))
	if err == nil {
		t.Fatal("want error for empty environment")
	}
	msg := err.Error()
	for _, want := range []string{"DATABASE_URL", "CMS_MASTER_SECRET"} {
		if !strings.Contains(msg, want) {
			t.Errorf("error %q must name %s (09-deployment.md §Configuration)", msg, want)
		}
	}
}

func TestBR_MEDIA_6_PartialS3ConfigFailsListingMissing(t *testing.T) {
	m := map[string]string{}
	for k, v := range minimalEnv {
		m[k] = v
	}
	m["S3_BUCKET"] = "media"
	_, err := LoadConfig(env(m))
	if err == nil {
		t.Fatal("partial S3_* group must fail startup (BR-MEDIA-6)")
	}
	msg := err.Error()
	for _, want := range []string{"S3_ENDPOINT", "S3_ACCESS_KEY", "S3_SECRET_KEY"} {
		if !strings.Contains(msg, want) {
			t.Errorf("error %q must list missing %s", msg, want)
		}
	}
}

func TestLoadConfig_FullS3GroupEnablesMedia(t *testing.T) {
	m := map[string]string{}
	for k, v := range minimalEnv {
		m[k] = v
	}
	m["S3_ENDPOINT"], m["S3_BUCKET"], m["S3_ACCESS_KEY"], m["S3_SECRET_KEY"] = "https://s3.example", "media", "ak", "sk"
	cfg, err := LoadConfig(env(m))
	if err != nil {
		t.Fatalf("unexpected error: %v", err)
	}
	if !cfg.MediaEnabled() {
		t.Error("full S3_* group must enable media")
	}
}

func TestLoadConfig_InvalidValuesCollected(t *testing.T) {
	m := map[string]string{}
	for k, v := range minimalEnv {
		m[k] = v
	}
	m["CMS_PORT"] = "not-a-port"
	m["CMS_LOG_LEVEL"] = "loud"
	m["CMS_TRUSTED_PROXY_CIDRS"] = "10.0.0.0/8,bogus"
	_, err := LoadConfig(env(m))
	if err == nil {
		t.Fatal("want error")
	}
	msg := err.Error()
	for _, want := range []string{"CMS_PORT", "CMS_LOG_LEVEL", "CMS_TRUSTED_PROXY_CIDRS"} {
		if !strings.Contains(msg, want) {
			t.Errorf("error %q must name %s", msg, want)
		}
	}
}
```

- [ ] **Step 4: Run tests to verify they fail**

Run: `go test ./internal/app/`
Expected: FAIL — `LoadConfig` undefined.

- [ ] **Step 5: Write `internal/app/config.go`**

```go
package app

import (
	"fmt"
	"log/slog"
	"net"
	"strconv"
	"strings"
)

// Config carries every value of the seventeen-variable table in
// BUSINESS_RULES.md §Naming Constants. It is parsed in full from Phase 1;
// later phases consume values but never reopen parsing (phasing spec,
// cross-phase rules).
type Config struct {
	DatabaseURL         string
	MasterSecret        string
	JWTPrivateKey       string // PEM; auto-generated when absent (BR-AUTH-10, P5)
	JWTPublicKey        string // derived from private when absent
	S3                  *S3Config // nil = media-less mode (BR-MEDIA-6)
	R2AccountID         string
	R2PublicBucketURL   string
	RecoveryEmail       string
	TrustedProxyCIDRs   []*net.IPNet
	EndUserRegistration bool
	Port                int
	LogLevel            slog.Level
	TrashRetentionDays  int
	RevisionLimit       int
}

type S3Config struct {
	Endpoint, Bucket, AccessKey, SecretKey string
}

func (c *Config) MediaEnabled() bool { return c.S3 != nil }

// ConfigError aggregates every problem so startup reports all of them at
// once, not the first (09-deployment.md §Configuration).
type ConfigError struct{ Problems []string }

func (e *ConfigError) Error() string {
	return "invalid configuration:\n  " + strings.Join(e.Problems, "\n  ")
}

var s3Vars = []string{"S3_ENDPOINT", "S3_BUCKET", "S3_ACCESS_KEY", "S3_SECRET_KEY"}

// LoadConfig reads the environment through lookup (os.LookupEnv in main;
// injected maps in tests).
func LoadConfig(lookup func(string) (string, bool)) (*Config, error) {
	var problems []string
	get := func(k string) string { v, _ := lookup(k); return v }

	cfg := &Config{
		DatabaseURL:       get("DATABASE_URL"),
		MasterSecret:      get("CMS_MASTER_SECRET"),
		JWTPrivateKey:     get("JWT_PRIVATE_KEY"),
		JWTPublicKey:      get("JWT_PUBLIC_KEY"),
		R2AccountID:       get("R2_ACCOUNT_ID"),
		R2PublicBucketURL: get("R2_PUBLIC_BUCKET_URL"),
		RecoveryEmail:     get("CMS_RECOVERY_EMAIL"),
	}

	if cfg.DatabaseURL == "" {
		problems = append(problems, "DATABASE_URL is required")
	}
	if cfg.MasterSecret == "" {
		problems = append(problems, "CMS_MASTER_SECRET is required")
	}

	// S3_* is optional as a unit: all four or none (BR-MEDIA-6).
	var present, missing []string
	for _, k := range s3Vars {
		if get(k) != "" {
			present = append(present, k)
		} else {
			missing = append(missing, k)
		}
	}
	switch {
	case len(present) == len(s3Vars):
		cfg.S3 = &S3Config{
			Endpoint:  get("S3_ENDPOINT"),
			Bucket:    get("S3_BUCKET"),
			AccessKey: get("S3_ACCESS_KEY"),
			SecretKey: get("S3_SECRET_KEY"),
		}
	case len(present) > 0:
		problems = append(problems, fmt.Sprintf(
			"the S3_* group is configured as a unit (BR-MEDIA-6): %s set but %s missing",
			strings.Join(present, ", "), strings.Join(missing, ", ")))
	}

	cfg.Port = 8080
	if v, ok := lookup("CMS_PORT"); ok {
		p, err := strconv.Atoi(v)
		if err != nil || p < 0 || p > 65535 {
			problems = append(problems, fmt.Sprintf("CMS_PORT %q is not a valid port", v))
		} else {
			cfg.Port = p
		}
	}

	cfg.LogLevel = slog.LevelInfo
	if v, ok := lookup("CMS_LOG_LEVEL"); ok {
		switch v {
		case "debug":
			cfg.LogLevel = slog.LevelDebug
		case "info":
			cfg.LogLevel = slog.LevelInfo
		case "warn":
			cfg.LogLevel = slog.LevelWarn
		case "error":
			cfg.LogLevel = slog.LevelError
		default:
			problems = append(problems, fmt.Sprintf(
				"CMS_LOG_LEVEL %q is not one of debug, info, warn, error", v))
		}
	}

	if v, ok := lookup("CMS_TRUSTED_PROXY_CIDRS"); ok && v != "" {
		for _, c := range strings.Split(v, ",") {
			_, ipnet, err := net.ParseCIDR(strings.TrimSpace(c))
			if err != nil {
				problems = append(problems, fmt.Sprintf(
					"CMS_TRUSTED_PROXY_CIDRS entry %q is not a valid CIDR", strings.TrimSpace(c)))
				continue
			}
			cfg.TrustedProxyCIDRs = append(cfg.TrustedProxyCIDRs, ipnet)
		}
	}

	if v, ok := lookup("CMS_END_USER_REGISTRATION"); ok {
		switch v {
		case "enabled":
			cfg.EndUserRegistration = true
		case "disabled", "":
			// default
		default:
			problems = append(problems, fmt.Sprintf(
				"CMS_END_USER_REGISTRATION %q is not enabled or disabled", v))
		}
	}

	cfg.TrashRetentionDays = intVar(lookup, "CMS_TRASH_RETENTION_DAYS", 30, &problems)
	cfg.RevisionLimit = intVar(lookup, "CMS_REVISION_LIMIT", 50, &problems)

	if len(problems) > 0 {
		return nil, &ConfigError{Problems: problems}
	}
	return cfg, nil
}

func intVar(lookup func(string) (string, bool), key string, def int, problems *[]string) int {
	v, ok := lookup(key)
	if !ok || v == "" {
		return def
	}
	n, err := strconv.Atoi(v)
	if err != nil || n < 1 {
		*problems = append(*problems, fmt.Sprintf("%s %q is not a positive integer", key, v))
		return def
	}
	return n
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `go test ./internal/app/ ./internal/access/`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add go.mod go.sum internal/access/ internal/app/
git commit -m "feat: go module, access principal model, seventeen-variable config loader"
```

---

### Task 3: Migration 0001, runner, sqlc, test database helper

**Files:**
- Create: `internal/store/migrations/0001_system_tables.sql`, `internal/store/migrate.go`, `internal/store/queries/users.sql`, `sqlc.yaml`, `internal/testdb/testdb.go`, `scripts/testdb.sh`
- Test: `internal/store/migrate_integration_test.go`
- Modify: `Makefile` (add `generate`, `unit`, `test` targets)

**Interfaces:**
- Consumes: nothing from earlier tasks (pure store layer).
- Produces:
  - `store.Migrate(ctx context.Context, pool *pgxpool.Pool, log *slog.Logger) (applied int, err error)`.
  - sqlc-generated `store.New(db store.DBTX) *store.Queries` with `(*Queries).CountUsers(ctx context.Context) (int64, error)`.
  - `testdb.URL(t *testing.T) string` — creates a uniquely named database from `TEST_DATABASE_URL`, drops it on cleanup, skips the test when `TEST_DATABASE_URL` is unset.
  - `testdb.Pool(t *testing.T) (*pgxpool.Pool, string)` — same, plus an open pool (closed on cleanup).

- [ ] **Step 1: Write `internal/store/migrations/0001_system_tables.sql`**

All 13 V1 system tables exactly per `07-data-model.md` §System Tables. IDs are UUID PKs generated application-side (UUIDv7) — no column defaults. Timestamps are app-supplied, so `NOT NULL` without defaults. FKs carry a named behavior only where 07 names one (admin deletion cascades sessions); everything else stays `NO ACTION` — later phases amend by forward migration if their delivery cycle needs more. `cms_reset_tokens.user_id` is polymorphic (`user_kind` discriminator) and cannot carry an FK; the admin-deletion cascade for reset tokens is application-level in P2.

```sql
-- 0001_system_tables.sql — the 13 V1 system tables (07-data-model.md).
-- Forward-only; recovery is restore-from-backup (10-project-structure.md §Migrations).

CREATE TABLE cms_users (
    id            UUID PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL CHECK (role IN ('super_admin','admin','editor','contributor','viewer')),
    created_at    TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL
);

CREATE TABLE cms_sessions (
    token_hash   TEXT PRIMARY KEY,
    user_id      UUID NOT NULL REFERENCES cms_users(id) ON DELETE CASCADE,
    csrf_hash    TEXT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL,
    last_seen_at TIMESTAMPTZ NOT NULL,
    ip           TEXT,
    user_agent   TEXT
);

CREATE TABLE cms_api_keys (
    id         UUID PRIMARY KEY,
    name       TEXT NOT NULL,
    token_hash TEXT NOT NULL UNIQUE,
    scopes     JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_by UUID NOT NULL REFERENCES cms_users(id),
    created_at TIMESTAMPTZ NOT NULL,
    revoked_at TIMESTAMPTZ
);

CREATE TABLE cms_end_users (
    id            UUID PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    disabled_at   TIMESTAMPTZ,
    created_at    TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL
);

CREATE TABLE cms_refresh_tokens (
    id          UUID PRIMARY KEY,
    family_id   UUID NOT NULL,
    end_user_id UUID NOT NULL REFERENCES cms_end_users(id),
    token_hash  TEXT NOT NULL UNIQUE,
    issued_at   TIMESTAMPTZ NOT NULL,
    rotated_at  TIMESTAMPTZ,
    revoked_at  TIMESTAMPTZ
);

CREATE TABLE cms_system_keys (
    name        TEXT PRIMARY KEY,
    private_pem TEXT NOT NULL,
    public_pem  TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL
);

CREATE TABLE cms_collections (
    id            UUID PRIMARY KEY,
    slug          TEXT NOT NULL UNIQUE,
    name          TEXT NOT NULL,
    access_rules  JSONB NOT NULL DEFAULT '{}'::jsonb,
    search_config JSONB,
    created_at    TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL
);

CREATE TABLE cms_fields (
    id            UUID PRIMARY KEY,
    collection_id UUID NOT NULL REFERENCES cms_collections(id),
    slug          TEXT NOT NULL,
    type          TEXT NOT NULL,
    config        JSONB NOT NULL DEFAULT '{}'::jsonb,
    position      INT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL,
    UNIQUE (collection_id, slug)
);

CREATE TABLE cms_revisions (
    id            UUID PRIMARY KEY,
    collection_id UUID NOT NULL REFERENCES cms_collections(id),
    record_id     UUID NOT NULL,
    version_no    BIGINT NOT NULL,
    data          JSONB NOT NULL,
    published     BOOLEAN NOT NULL DEFAULT false,
    created_by    UUID NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL,
    UNIQUE (collection_id, record_id, version_no)
);

-- At most one published revision per record (07 §System Tables).
CREATE UNIQUE INDEX ix_cms_revisions_published
    ON cms_revisions (collection_id, record_id) WHERE published;

CREATE TABLE cms_media (
    id           UUID PRIMARY KEY,
    object_key   TEXT NOT NULL UNIQUE,
    mime         TEXT NOT NULL,
    size_bytes   BIGINT NOT NULL,
    metadata     JSONB NOT NULL DEFAULT '{}'::jsonb,
    status       TEXT NOT NULL CHECK (status IN ('pending','finalized')),
    created_by   UUID NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL,
    finalized_at TIMESTAMPTZ
);

CREATE TABLE cms_reset_tokens (
    id         UUID PRIMARY KEY,
    user_kind  TEXT NOT NULL CHECK (user_kind IN ('admin','end_user')),
    user_id    UUID NOT NULL,
    token_hash TEXT NOT NULL UNIQUE,
    expires_at TIMESTAMPTZ NOT NULL,
    used_at    TIMESTAMPTZ,
    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE cms_idempotency_keys (
    key_hash     TEXT NOT NULL,
    principal_id UUID NOT NULL,
    record_id    UUID NOT NULL,
    request_hash TEXT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL,
    UNIQUE (key_hash, principal_id)
);

CREATE TABLE cms_media_deletions (
    object_key TEXT PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL
);
```

- [ ] **Step 2: Write `internal/store/migrate.go`**

```go
// Package store owns system-table access: embedded migrations (this file)
// and sqlc-generated queries. Collection tables never appear here —
// they belong to query.Builder exclusively (BR-SCHEMA-3, arrives P4).
package store

import (
	"context"
	"embed"
	"fmt"
	"log/slog"
	"sort"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

//go:embed migrations/*.sql
var migrationsFS embed.FS

// migrationLockKey is the fixed migration advisory key (09-deployment.md
// §Startup): a concurrently restarting replacement waits rather than racing.
const migrationLockKey = 0x636D7300

// Migrate applies pending embedded migrations in filename order under a
// session advisory lock, recording each in cms_migrations. Idempotent:
// applied versions are skipped. Returns the number applied.
func Migrate(ctx context.Context, pool *pgxpool.Pool, log *slog.Logger) (int, error) {
	conn, err := pool.Acquire(ctx)
	if err != nil {
		return 0, fmt.Errorf("migrate: acquire connection: %w", err)
	}
	defer conn.Release()

	if _, err := conn.Exec(ctx, "SELECT pg_advisory_lock($1)", migrationLockKey); err != nil {
		return 0, fmt.Errorf("migrate: advisory lock: %w", err)
	}
	defer conn.Exec(context.WithoutCancel(ctx), "SELECT pg_advisory_unlock($1)", migrationLockKey)

	if _, err := conn.Exec(ctx, `CREATE TABLE IF NOT EXISTS cms_migrations (
		version    INT PRIMARY KEY,
		name       TEXT NOT NULL,
		applied_at TIMESTAMPTZ NOT NULL DEFAULT now()
	)`); err != nil {
		return 0, fmt.Errorf("migrate: create ledger: %w", err)
	}

	entries, err := migrationsFS.ReadDir("migrations")
	if err != nil {
		return 0, fmt.Errorf("migrate: read embedded FS: %w", err)
	}
	names := make([]string, 0, len(entries))
	for _, e := range entries {
		names = append(names, e.Name())
	}
	sort.Strings(names)

	applied := 0
	for _, name := range names {
		var version int
		if _, err := fmt.Sscanf(name, "%d_", &version); err != nil {
			return applied, fmt.Errorf("migrate: %s does not match NNNN_description.sql", name)
		}
		var exists bool
		if err := conn.QueryRow(ctx,
			"SELECT EXISTS(SELECT 1 FROM cms_migrations WHERE version = $1)", version,
		).Scan(&exists); err != nil {
			return applied, fmt.Errorf("migrate: check version %d: %w", version, err)
		}
		if exists {
			continue
		}

		sqlBytes, err := migrationsFS.ReadFile("migrations/" + name)
		if err != nil {
			return applied, fmt.Errorf("migrate: read %s: %w", name, err)
		}
		start := time.Now()
		tx, err := conn.Begin(ctx)
		if err != nil {
			return applied, fmt.Errorf("migrate: begin %s: %w", name, err)
		}
		if _, err := tx.Exec(ctx, string(sqlBytes)); err != nil {
			tx.Rollback(ctx)
			return applied, fmt.Errorf("migrate: apply %s: %w", name, err)
		}
		if _, err := tx.Exec(ctx,
			"INSERT INTO cms_migrations (version, name) VALUES ($1, $2)", version, name,
		); err != nil {
			tx.Rollback(ctx)
			return applied, fmt.Errorf("migrate: record %s: %w", name, err)
		}
		if err := tx.Commit(ctx); err != nil {
			return applied, fmt.Errorf("migrate: commit %s: %w", name, err)
		}
		applied++
		log.Info("migration applied", "version", version, "name", name,
			"duration_ms", time.Since(start).Milliseconds())
	}
	return applied, nil
}
```

- [ ] **Step 3: Write the sqlc source and config**

`internal/store/queries/users.sql`:

```sql
-- name: CountUsers :one
SELECT count(*) FROM cms_users;
```

`sqlc.yaml` (repo root):

```yaml
version: "2"
sql:
  - engine: "postgresql"
    schema: "internal/store/migrations"
    queries: "internal/store/queries"
    gen:
      go:
        package: "store"
        out: "internal/store"
        sql_package: "pgx/v5"
```

Install sqlc as a Go tool and generate:

```bash
go get -tool github.com/sqlc-dev/sqlc/cmd/sqlc@latest
go tool sqlc generate
```

Expected: `internal/store/db.go`, `internal/store/models.go`, `internal/store/users.sql.go` appear. Generated code commits to the repo (10-project-structure §Code Generation).

- [ ] **Step 4: Write `internal/testdb/testdb.go`**

```go
// Package testdb provisions one throwaway database per test from the
// disposable postgres:16 container that scripts/testdb.sh starts.
// Used only by _test.go files.
package testdb

import (
	"context"
	"fmt"
	"math/rand"
	"net/url"
	"os"
	"testing"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

// URL creates a uniquely named database and returns its connection string.
// The database is dropped on test cleanup. Skips when TEST_DATABASE_URL is
// unset (unit-only runs).
func URL(t *testing.T) string {
	t.Helper()
	base := os.Getenv("TEST_DATABASE_URL")
	if base == "" {
		t.Skip("TEST_DATABASE_URL not set; run via make test")
	}
	ctx := context.Background()
	admin, err := pgx.Connect(ctx, base)
	if err != nil {
		t.Fatalf("testdb: connect admin: %v", err)
	}
	name := fmt.Sprintf("cms_test_%08x", rand.Uint32())
	if _, err := admin.Exec(ctx, "CREATE DATABASE "+name); err != nil {
		admin.Close(ctx)
		t.Fatalf("testdb: create %s: %v", name, err)
	}
	admin.Close(ctx)

	t.Cleanup(func() {
		ctx := context.Background()
		admin, err := pgx.Connect(ctx, base)
		if err != nil {
			return
		}
		defer admin.Close(ctx)
		admin.Exec(ctx, "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = $1", name)
		admin.Exec(ctx, "DROP DATABASE IF EXISTS "+name)
	})

	u, err := url.Parse(base)
	if err != nil {
		t.Fatalf("testdb: parse TEST_DATABASE_URL: %v", err)
	}
	u.Path = "/" + name
	return u.String()
}

// Pool returns an open pool on a fresh throwaway database plus its URL.
func Pool(t *testing.T) (*pgxpool.Pool, string) {
	t.Helper()
	dbURL := URL(t)
	pool, err := pgxpool.New(context.Background(), dbURL)
	if err != nil {
		t.Fatalf("testdb: open pool: %v", err)
	}
	t.Cleanup(pool.Close)
	return pool, dbURL
}
```

- [ ] **Step 5: Write `scripts/testdb.sh`** (make it executable)

```bash
#!/usr/bin/env bash
# Starts a disposable postgres:16, exports TEST_DATABASE_URL, runs the given
# command, and always stops the container (run-and-verify skill contract).
set -euo pipefail
name="cms-test-pg-$$"
docker run --rm -d --name "$name" \
  -e POSTGRES_PASSWORD=cms -e POSTGRES_DB=cms \
  -p 127.0.0.1:0:5432 postgres:16 >/dev/null
trap 'docker stop "$name" >/dev/null 2>&1 || true' EXIT
port=$(docker inspect -f '{{ (index (index .NetworkSettings.Ports "5432/tcp") 0).HostPort }}' "$name")
export TEST_DATABASE_URL="postgres://postgres:cms@127.0.0.1:${port}/cms?sslmode=disable"
for _ in $(seq 1 120); do
  docker exec "$name" pg_isready -U postgres >/dev/null 2>&1 && break
  sleep 0.5
done
"$@"
```

```bash
chmod +x scripts/testdb.sh
```

- [ ] **Step 6: Write the failing integration test**

`internal/store/migrate_integration_test.go`:

```go
//go:build integration

package store_test

import (
	"context"
	"log/slog"
	"testing"

	"github.com/codebabbler/golang-cms/internal/store"
	"github.com/codebabbler/golang-cms/internal/testdb"
)

var systemTables = []string{
	"cms_users", "cms_sessions", "cms_api_keys", "cms_end_users",
	"cms_refresh_tokens", "cms_system_keys", "cms_collections", "cms_fields",
	"cms_revisions", "cms_media", "cms_reset_tokens", "cms_idempotency_keys",
	"cms_media_deletions", "cms_migrations",
}

func TestMigrate_CreatesAllSystemTablesAndIsIdempotent(t *testing.T) {
	pool, _ := testdb.Pool(t)
	ctx := context.Background()

	applied, err := store.Migrate(ctx, pool, slog.Default())
	if err != nil {
		t.Fatalf("first run: %v", err)
	}
	if applied != 1 {
		t.Errorf("first run applied = %d, want 1", applied)
	}
	for _, tbl := range systemTables {
		var exists bool
		err := pool.QueryRow(ctx,
			"SELECT EXISTS(SELECT 1 FROM information_schema.tables WHERE table_name = $1)", tbl,
		).Scan(&exists)
		if err != nil || !exists {
			t.Errorf("table %s missing after migrate (err=%v)", tbl, err)
		}
	}

	applied, err = store.Migrate(ctx, pool, slog.Default())
	if err != nil {
		t.Fatalf("second run: %v", err)
	}
	if applied != 0 {
		t.Errorf("second run applied = %d, want 0 (idempotent)", applied)
	}
}

func TestBR_RBAC_1_RoleCheckConstraintRejectsSixthRole(t *testing.T) {
	pool, _ := testdb.Pool(t)
	ctx := context.Background()
	if _, err := store.Migrate(ctx, pool, slog.Default()); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	_, err := pool.Exec(ctx, `INSERT INTO cms_users (id, email, password_hash, role, created_at, updated_at)
		VALUES (gen_random_uuid(), 'x@example.com', 'h', 'owner', now(), now())`)
	if err == nil {
		t.Fatal("role outside the five (BR-RBAC-1) must violate the CHECK constraint")
	}
}

func TestCountUsers_ZeroOnFreshDatabase(t *testing.T) {
	pool, _ := testdb.Pool(t)
	ctx := context.Background()
	if _, err := store.Migrate(ctx, pool, slog.Default()); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	n, err := store.New(pool).CountUsers(ctx)
	if err != nil {
		t.Fatalf("CountUsers: %v", err)
	}
	if n != 0 {
		t.Errorf("CountUsers = %d, want 0", n)
	}
}
```

- [ ] **Step 7: Add test targets to the Makefile**

Add to `Makefile` (keep Task 1 content):

```make
generate:
	go tool sqlc generate

unit: web/dist/index.html
	go test ./...

test: web/dist/index.html
	./scripts/testdb.sh go test -tags integration ./...
```

Note: `go test` compiles `web/embed.go` once Task 6 lands, hence the `web/dist` dependency — harmless now, required later.

- [ ] **Step 8: Run the integration tests**

Run: `make test`
Expected: PASS (3 tests in store, config tests from Task 2 also run).

- [ ] **Step 9: Commit**

```bash
git add sqlc.yaml scripts/testdb.sh internal/store/ internal/testdb/ Makefile go.mod go.sum
git commit -m "feat: migration 0001 (13 system tables), embedded runner with cms_migrations ledger, sqlc scaffold"
```

---

### Task 4: Audit package

**Files:**
- Create: `internal/audit/audit.go`
- Test: `internal/audit/audit_test.go`

**Interfaces:**
- Consumes: `access.Principal`, `access.Kind` (Task 2).
- Produces:
  - `audit.EntityRef` (string type, e.g. `"collection:posts/field:summary"`).
  - `audit.Event{Actor access.Principal; Action string; Entity EntityRef; Detail map[string]any; At time.Time}` (02 §audit.Recorder).
  - `audit.Sink interface{ Write(Event) error }`.
  - `audit.Recorder` with `Emit(ctx context.Context, e Event)` — stamps `At` when zero, resolves the request ID from ctx. (02's schematic omits ctx; it is required to carry the 08 §Audit `request_id` correlation.)
  - `audit.NewRecorder(sink Sink, log *slog.Logger) *Recorder`; `audit.NewSlogSink(log *slog.Logger) Sink`.
  - `audit.WithRequestID(ctx, id string) context.Context`, `audit.RequestIDFromContext(ctx) string` — the single request-correlation context key; httpapi's RequestID middleware sets it (Task 5), and later-phase services read it without importing httpapi.

- [ ] **Step 1: Write the failing test**

`internal/audit/audit_test.go`:

```go
package audit

import (
	"bytes"
	"context"
	"encoding/json"
	"log/slog"
	"testing"

	"github.com/codebabbler/golang-cms/internal/access"
	"github.com/google/uuid"
)

func TestBR_AUDIT_1_EmitWritesActorActionEntityTimestamp(t *testing.T) {
	var buf bytes.Buffer
	log := slog.New(slog.NewJSONHandler(&buf, nil))
	rec := NewRecorder(NewSlogSink(log), log)

	actorID := uuid.New()
	ctx := WithRequestID(context.Background(), "req-123")
	rec.Emit(ctx, Event{
		Actor:  access.Principal{Kind: access.KindAdmin, ID: actorID, Role: access.RoleAdmin},
		Action: "schema.collection.create",
		Entity: EntityRef("collection:posts"),
		Detail: map[string]any{"slug": "posts"},
	})

	var line map[string]any
	if err := json.Unmarshal(buf.Bytes(), &line); err != nil {
		t.Fatalf("audit line is not JSON: %v (%q)", err, buf.String())
	}
	// Line shape per 08-observability.md §Audit Event Stream.
	t.Run("BR-AUDIT-2 slog sink shape", func(t *testing.T) {
		checks := map[string]string{
			"msg":        "audit",
			"actor_kind": "admin",
			"actor_id":   actorID.String(),
			"action":     "schema.collection.create",
			"entity":     "collection:posts",
			"request_id": "req-123",
		}
		for k, want := range checks {
			if got, _ := line[k].(string); got != want {
				t.Errorf("%s = %q, want %q", k, got, want)
			}
		}
		if _, ok := line["at"].(string); !ok {
			t.Error("at timestamp missing")
		}
		if _, ok := line["detail"]; !ok {
			t.Error("detail missing")
		}
	})
}
```

- [ ] **Step 2: Run to verify it fails**

Run: `go test ./internal/audit/`
Expected: FAIL — package does not exist.

- [ ] **Step 3: Write `internal/audit/audit.go`**

```go
// Package audit is the mutation event stream (BR-AUDIT-1). V1 configures the
// slog sink; V2 adds the cms_audit_log sink behind the same interface without
// touching call sites (BR-AUDIT-2).
package audit

import (
	"context"
	"log/slog"
	"time"

	"github.com/codebabbler/golang-cms/internal/access"
)

// EntityRef names the acted-on entity, e.g. "collection:posts/field:summary".
type EntityRef string

// Event per 02-core-interfaces.md §audit.Recorder. Action follows the closed
// domain.entity.verb vocabulary (08-observability.md); the vocabulary constant
// list grows with each subsystem's phase.
type Event struct {
	Actor  access.Principal
	Action string
	Entity EntityRef
	Detail map[string]any
	At     time.Time
}

type Sink interface {
	Write(Event) error
}

type Recorder struct {
	sink Sink
	log  *slog.Logger
}

func NewRecorder(sink Sink, log *slog.Logger) *Recorder {
	return &Recorder{sink: sink, log: log}
}

// Emit stamps At when zero and forwards to the sink. A sink failure logs at
// error and never fails the caller's mutation — the audit stream is telemetry,
// not a transaction participant.
func (r *Recorder) Emit(ctx context.Context, e Event) {
	if e.At.IsZero() {
		e.At = time.Now().UTC()
	}
	if e.Detail == nil {
		e.Detail = map[string]any{}
	}
	if e.Detail["request_id"] == nil {
		if id := RequestIDFromContext(ctx); id != "" {
			e.Detail["request_id"] = id
		}
	}
	if err := r.sink.Write(e); err != nil {
		r.log.Error("audit sink write failed", "action", e.Action, "err", err)
	}
}

type slogSink struct{ log *slog.Logger }

// NewSlogSink returns the V1 sink: one distinguished slog line per event
// (08-observability.md §Audit Event Stream).
func NewSlogSink(log *slog.Logger) Sink { return &slogSink{log: log} }

func (s *slogSink) Write(e Event) error {
	requestID, _ := e.Detail["request_id"].(string)
	detail := make(map[string]any, len(e.Detail))
	for k, v := range e.Detail {
		if k != "request_id" {
			detail[k] = v
		}
	}
	s.log.Info("audit",
		"request_id", requestID,
		"actor_kind", string(e.Actor.Kind),
		"actor_id", e.Actor.ID.String(),
		"action", e.Action,
		"entity", string(e.Entity),
		"detail", detail,
		"at", e.At.Format(time.RFC3339Nano),
	)
	return nil
}

type ctxKey struct{}

// WithRequestID attaches the request correlation ID. httpapi.RequestID sets
// it; every audit event and downstream consumer reads it from here.
func WithRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, ctxKey{}, id)
}

func RequestIDFromContext(ctx context.Context) string {
	id, _ := ctx.Value(ctxKey{}).(string)
	return id
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `go test ./internal/audit/`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add internal/audit/
git commit -m "feat: audit recorder interface with slog sink and request-id correlation"
```

---

### Task 5: httpapi core — envelope, error registry, middleware, health, router

**Files:**
- Create: `internal/httpapi/errors.go`, `internal/httpapi/envelope.go`, `internal/httpapi/middleware.go`, `internal/httpapi/health.go`, `internal/httpapi/router.go`
- Test: `internal/httpapi/errors_test.go`, `internal/httpapi/middleware_test.go`, `internal/httpapi/health_test.go`

**Interfaces:**
- Consumes: `audit.WithRequestID`, `audit.RequestIDFromContext` (Task 4).
- Produces:
  - `httpapi.ErrorCode` + nine constants (`CodeValidationFailed`, `CodeUnauthorized`, `CodeForbidden`, `CodeNotFound`, `CodeConflict`, `CodeRateLimited`, `CodePayloadTooLarge`, `CodeInternal`, `CodeUnavailable`).
  - `httpapi.WriteError(w http.ResponseWriter, r *http.Request, code ErrorCode, message string, details any)` — details may be nil; the sole error-writing path (BR-API-3).
  - `httpapi.WriteData(w http.ResponseWriter, status int, data any, meta any)`.
  - `httpapi.RequestID`, `httpapi.Logger(log *slog.Logger)`, `httpapi.Recover(log *slog.Logger)` middleware (each `func(http.Handler) http.Handler`).
  - `httpapi.NewHealthState() *HealthState` with `SetDraining(bool)`, `Draining() bool`; `httpapi.Pinger interface{ Ping(context.Context) error }`.
  - `httpapi.NewRouter(cfg RouterConfig) http.Handler`; `RouterConfig{Log *slog.Logger; DB Pinger; Health *HealthState; SPA fs.FS}` (SPA nil-safe until Task 6 wires it).

- [ ] **Step 1: Write the failing registry test**

`internal/httpapi/errors_test.go`:

```go
package httpapi

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestBR_API_3_RegistryIsClosedAndStatusesMatch04(t *testing.T) {
	want := map[ErrorCode]int{
		CodeValidationFailed: 422,
		CodeUnauthorized:     401,
		CodeForbidden:        403,
		CodeNotFound:         404,
		CodeConflict:         409,
		CodeRateLimited:      429,
		CodePayloadTooLarge:  413,
		CodeInternal:         500,
		CodeUnavailable:      503,
	}
	if len(errorStatus) != len(want) {
		t.Fatalf("registry has %d codes, want exactly %d (closed registry, 04-api-layer.md)", len(errorStatus), len(want))
	}
	for code, status := range want {
		if errorStatus[code] != status {
			t.Errorf("%s → %d, want %d", code, errorStatus[code], status)
		}
	}
}

func TestWriteError_EnvelopeShape(t *testing.T) {
	rr := httptest.NewRecorder()
	r := httptest.NewRequest(http.MethodGet, "/x", nil)
	WriteError(rr, r, CodeConflict, "version mismatch", []map[string]any{{"field": "version"}})

	if rr.Code != 409 {
		t.Errorf("status = %d, want 409", rr.Code)
	}
	if ct := rr.Header().Get("Content-Type"); ct != "application/json; charset=utf-8" {
		t.Errorf("Content-Type = %q", ct)
	}
	var body struct {
		Error struct {
			Code    string          `json:"code"`
			Message string          `json:"message"`
			Details json.RawMessage `json:"details"`
		} `json:"error"`
	}
	if err := json.Unmarshal(rr.Body.Bytes(), &body); err != nil {
		t.Fatalf("not JSON: %v", err)
	}
	if body.Error.Code != "conflict" || body.Error.Message != "version mismatch" || body.Error.Details == nil {
		t.Errorf("unexpected envelope: %s", rr.Body.String())
	}
}

func TestWriteData_Envelope(t *testing.T) {
	rr := httptest.NewRecorder()
	WriteData(rr, http.StatusOK, map[string]string{"k": "v"}, nil)
	var body struct {
		Data map[string]string `json:"data"`
	}
	if err := json.Unmarshal(rr.Body.Bytes(), &body); err != nil || body.Data["k"] != "v" {
		t.Fatalf("unexpected envelope: %s (err=%v)", rr.Body.String(), err)
	}
}
```

- [ ] **Step 2: Run to verify failure**

Run: `go test ./internal/httpapi/`
Expected: FAIL — package does not exist.

- [ ] **Step 3: Write `internal/httpapi/errors.go` and `envelope.go`**

`internal/httpapi/errors.go`:

```go
package httpapi

import (
	"encoding/json"
	"net/http"
)

// ErrorCode is the closed error registry of 04-api-layer.md. Adding a code
// requires a BUSINESS_RULES review (BR-API-3).
type ErrorCode string

const (
	CodeValidationFailed ErrorCode = "validation_failed"
	CodeUnauthorized     ErrorCode = "unauthorized"
	CodeForbidden        ErrorCode = "forbidden"
	CodeNotFound         ErrorCode = "not_found"
	CodeConflict         ErrorCode = "conflict"
	CodeRateLimited      ErrorCode = "rate_limited"
	CodePayloadTooLarge  ErrorCode = "payload_too_large"
	CodeInternal         ErrorCode = "internal"
	CodeUnavailable      ErrorCode = "unavailable"
)

var errorStatus = map[ErrorCode]int{
	CodeValidationFailed: http.StatusUnprocessableEntity,
	CodeUnauthorized:     http.StatusUnauthorized,
	CodeForbidden:        http.StatusForbidden,
	CodeNotFound:         http.StatusNotFound,
	CodeConflict:         http.StatusConflict,
	CodeRateLimited:      http.StatusTooManyRequests,
	CodePayloadTooLarge:  http.StatusRequestEntityTooLarge,
	CodeInternal:         http.StatusInternalServerError,
	CodeUnavailable:      http.StatusServiceUnavailable,
}

type errorBody struct {
	Error errorObject `json:"error"`
}

type errorObject struct {
	Code    ErrorCode `json:"code"`
	Message string    `json:"message"`
	Details any       `json:"details,omitempty"`
}

// WriteError is the only path that writes error responses (BR-API-3).
// An unregistered code is a programmer error and downgrades to internal.
func WriteError(w http.ResponseWriter, r *http.Request, code ErrorCode, message string, details any) {
	status, ok := errorStatus[code]
	if !ok {
		code, status, message, details = CodeInternal, http.StatusInternalServerError, "internal error", nil
	}
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(errorBody{Error: errorObject{Code: code, Message: message, Details: details}})
}
```

`internal/httpapi/envelope.go`:

```go
package httpapi

import (
	"encoding/json"
	"net/http"
)

type dataEnvelope struct {
	Data any `json:"data"`
	Meta any `json:"meta,omitempty"`
}

// WriteData writes the success envelope of 04-api-layer.md.
func WriteData(w http.ResponseWriter, status int, data any, meta any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(dataEnvelope{Data: data, Meta: meta})
}
```

- [ ] **Step 4: Write the failing middleware/health tests**

`internal/httpapi/middleware_test.go`:

```go
package httpapi

import (
	"bytes"
	"encoding/json"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/codebabbler/golang-cms/internal/audit"
	"github.com/go-chi/chi/v5"
	"github.com/google/uuid"
)

func TestRequestID_SetsHeaderAndContext(t *testing.T) {
	var gotCtxID string
	h := RequestID(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		gotCtxID = audit.RequestIDFromContext(r.Context())
	}))
	rr := httptest.NewRecorder()
	h.ServeHTTP(rr, httptest.NewRequest(http.MethodGet, "/", nil))

	headerID := rr.Header().Get("X-Request-ID")
	if headerID == "" || headerID != gotCtxID {
		t.Fatalf("header %q and context %q must both carry the ID", headerID, gotCtxID)
	}
	if _, err := uuid.Parse(headerID); err != nil {
		t.Errorf("request ID %q is not a UUID: %v", headerID, err)
	}
}

func TestLogger_OneLinePerRequestWithChiPattern(t *testing.T) {
	var buf bytes.Buffer
	log := slog.New(slog.NewJSONHandler(&buf, nil))
	r := chi.NewRouter()
	r.Use(RequestID, Logger(log))
	r.Get("/things/{id}", func(w http.ResponseWriter, r *http.Request) { w.Write([]byte("ok")) })

	rr := httptest.NewRecorder()
	r.ServeHTTP(rr, httptest.NewRequest(http.MethodGet, "/things/42", nil))

	var line map[string]any
	if err := json.Unmarshal(buf.Bytes(), &line); err != nil {
		t.Fatalf("log line not JSON: %v (%q)", err, buf.String())
	}
	if line["msg"] != "request" {
		t.Errorf("msg = %v, want request", line["msg"])
	}
	if line["route"] != "/things/{id}" {
		t.Errorf("route = %v, want chi pattern /things/{id} — raw paths only at debug (08)", line["route"])
	}
	for _, k := range []string{"request_id", "method", "status", "duration_ms", "bytes"} {
		if _, ok := line[k]; !ok {
			t.Errorf("log line missing %s", k)
		}
	}
}

func TestRecover_PanicReturnsInternalEnvelope(t *testing.T) {
	var buf bytes.Buffer
	log := slog.New(slog.NewJSONHandler(&buf, nil))
	h := Recover(log)(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		panic("boom")
	}))
	rr := httptest.NewRecorder()
	h.ServeHTTP(rr, httptest.NewRequest(http.MethodGet, "/", nil))

	if rr.Code != 500 {
		t.Errorf("status = %d, want 500", rr.Code)
	}
	var body struct {
		Error struct{ Code string `json:"code"` } `json:"error"`
	}
	json.Unmarshal(rr.Body.Bytes(), &body)
	if body.Error.Code != "internal" {
		t.Errorf("code = %q, want internal", body.Error.Code)
	}
	if !bytes.Contains(buf.Bytes(), []byte("boom")) {
		t.Error("panic value must be logged at error with stack")
	}
}
```

`internal/httpapi/health_test.go`:

```go
package httpapi

import (
	"context"
	"errors"
	"io/fs"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"testing"
	"testing/fstest"
)

type fakePinger struct{ err error }

func (f fakePinger) Ping(context.Context) error { return f.err }

func emptySPA() fs.FS {
	return fstest.MapFS{"index.html": &fstest.MapFile{Data: []byte("<html></html>")}}
}

func newTestRouter(p Pinger, h *HealthState) http.Handler {
	return NewRouter(RouterConfig{
		Log:    slog.New(slog.DiscardHandler),
		DB:     p,
		Health: h,
		SPA:    emptySPA(),
	})
}

func TestHealthEndpoints(t *testing.T) {
	health := NewHealthState()
	router := newTestRouter(fakePinger{}, health)

	get := func(path string) int {
		rr := httptest.NewRecorder()
		router.ServeHTTP(rr, httptest.NewRequest(http.MethodGet, path, nil))
		return rr.Code
	}

	if c := get("/healthz"); c != 200 {
		t.Errorf("healthz = %d, want 200", c)
	}
	if c := get("/readyz"); c != 200 {
		t.Errorf("readyz = %d, want 200", c)
	}

	t.Run("BR-RUNTIME-6 drain flips both to 503 (EC-14)", func(t *testing.T) {
		health.SetDraining(true)
		if c := get("/healthz"); c != 503 {
			t.Errorf("draining healthz = %d, want 503", c)
		}
		if c := get("/readyz"); c != 503 {
			t.Errorf("draining readyz = %d, want 503", c)
		}
		health.SetDraining(false)
	})

	t.Run("readyz 503 on db failure", func(t *testing.T) {
		router := newTestRouter(fakePinger{err: errors.New("down")}, NewHealthState())
		rr := httptest.NewRecorder()
		router.ServeHTTP(rr, httptest.NewRequest(http.MethodGet, "/readyz", nil))
		if rr.Code != 503 {
			t.Errorf("readyz = %d, want 503", rr.Code)
		}
	})
}
```

- [ ] **Step 5: Run to verify failure**

Run: `go test ./internal/httpapi/`
Expected: FAIL — middleware and router undefined.

- [ ] **Step 6: Write `internal/httpapi/middleware.go`**

```go
package httpapi

import (
	"context"
	"log/slog"
	"net/http"
	"runtime/debug"
	"time"

	"github.com/codebabbler/golang-cms/internal/audit"
	"github.com/go-chi/chi/v5"
	"github.com/google/uuid"
)

// requestDeadline and defaultBodyCap are compiled-in constants
// (09-deployment.md §Timeouts).
const (
	requestDeadline = 25 * time.Second
	defaultBodyCap  = 64 << 10 // 64 KiB; the 5 MiB record-write class arrives P4
)

// RequestID assigns a UUIDv7 per request, returns it as X-Request-ID, and
// stores it in context via the audit package's correlation key (08).
func RequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := uuid.Must(uuid.NewV7()).String()
		w.Header().Set("X-Request-ID", id)
		next.ServeHTTP(w, r.WithContext(audit.WithRequestID(r.Context(), id)))
	})
}

type statusWriter struct {
	http.ResponseWriter
	status int
	bytes  int
}

func (sw *statusWriter) WriteHeader(code int) {
	if sw.status == 0 {
		sw.status = code
	}
	sw.ResponseWriter.WriteHeader(code)
}

func (sw *statusWriter) Write(b []byte) (int, error) {
	if sw.status == 0 {
		sw.status = http.StatusOK
	}
	n, err := sw.ResponseWriter.Write(b)
	sw.bytes += n
	return n, err
}

// Logger emits one line per request at completion (08 §Logging Conventions).
// Routes log as chi patterns, never raw paths, at info.
func Logger(log *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			sw := &statusWriter{ResponseWriter: w}
			next.ServeHTTP(sw, r)

			route := "(unmatched)"
			if rc := chi.RouteContext(r.Context()); rc != nil {
				if p := rc.RoutePattern(); p != "" {
					route = p
				}
			}
			log.LogAttrs(r.Context(), slog.LevelInfo, "request",
				slog.String("request_id", audit.RequestIDFromContext(r.Context())),
				slog.String("method", r.Method),
				slog.String("route", route),
				slog.Int("status", sw.status),
				slog.Int64("duration_ms", time.Since(start).Milliseconds()),
				slog.Int("bytes", sw.bytes),
			)
		})
	}
}

// Recover logs recovered panics at error with the stack and request ID, then
// returns the internal envelope (08 §Logging Conventions).
func Recover(log *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if rec := recover(); rec != nil {
					log.Error("panic recovered",
						"request_id", audit.RequestIDFromContext(r.Context()),
						"panic", rec,
						"stack", string(debug.Stack()),
					)
					WriteError(w, r, CodeInternal, "internal error", nil)
				}
			}()
			next.ServeHTTP(w, r)
		})
	}
}

// deadline applies the per-request context deadline.
func deadline(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), requestDeadline)
		defer cancel()
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

// bodyCap enforces the default request-body cap.
func bodyCap(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Body != nil {
			r.Body = http.MaxBytesReader(w, r.Body, defaultBodyCap)
		}
		next.ServeHTTP(w, r)
	})
}
```

- [ ] **Step 7: Write `internal/httpapi/health.go` and `router.go`**

`internal/httpapi/health.go`:

```go
package httpapi

import (
	"context"
	"net/http"
	"sync/atomic"
)

// Pinger is what /readyz probes — satisfied by *pgxpool.Pool.
type Pinger interface {
	Ping(ctx context.Context) error
}

// HealthState carries the drain flag: EC-14 step 1 flips /readyz and
// /healthz to 503 so the proxy stops routing new work.
type HealthState struct{ draining atomic.Bool }

func NewHealthState() *HealthState            { return &HealthState{} }
func (h *HealthState) SetDraining(v bool)     { h.draining.Store(v) }
func (h *HealthState) Draining() bool         { return h.draining.Load() }

func healthzHandler(h *HealthState) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if h.Draining() {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
	}
}

func readyzHandler(h *HealthState, db Pinger) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		if h.Draining() || db.Ping(r.Context()) != nil {
			w.WriteHeader(http.StatusServiceUnavailable)
			return
		}
		w.WriteHeader(http.StatusOK)
	}
}
```

`internal/httpapi/router.go`:

```go
package httpapi

import (
	"io/fs"
	"log/slog"
	"net/http"

	"github.com/go-chi/chi/v5"
)

type RouterConfig struct {
	Log    *slog.Logger
	DB     Pinger
	Health *HealthState
	SPA    fs.FS // built web/dist; served with the EC-15 cache contract
}

// NewRouter assembles the P1 surface in the normative middleware order
// prefix (04 §Middleware Order): RequestID → Logger → Recover. RateLimit
// and Auth slot into their documented positions in P2/P5/P6.
func NewRouter(cfg RouterConfig) http.Handler {
	r := chi.NewRouter()
	r.Use(RequestID, Logger(cfg.Log), Recover(cfg.Log), deadline, bodyCap)

	r.Get("/healthz", healthzHandler(cfg.Health))
	r.Get("/readyz", readyzHandler(cfg.Health, cfg.DB))

	// Unknown /api paths answer through WriteError (BR-API-3); the SPA
	// fallback must not swallow them.
	r.Route("/api", func(api chi.Router) {
		api.NotFound(func(w http.ResponseWriter, r *http.Request) {
			WriteError(w, r, CodeNotFound, "no such route", nil)
		})
		api.MethodNotAllowed(func(w http.ResponseWriter, r *http.Request) {
			WriteError(w, r, CodeNotFound, "no such route", nil)
		})
	})

	if cfg.SPA != nil {
		r.NotFound(spaHandler(cfg.SPA))
	}
	return r
}
```

Note: `spaHandler` arrives in Task 6 — for this task, add a temporary stub at the bottom of `router.go` so the package compiles, replaced in Task 6:

```go
// spaHandler is implemented in Task 6 (spa.go); temporary stub.
func spaHandler(dist fs.FS) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) { http.NotFound(w, r) }
}
```

- [ ] **Step 8: Run tests** (the health test's SPA index assertion is Task 6's; here `/healthz`, `/readyz`, envelope, middleware must pass)

Run: `go test ./internal/httpapi/`
Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add internal/httpapi/ go.mod go.sum
git commit -m "feat: httpapi skeleton — closed error registry, envelope, request middleware, health endpoints"
```

---

### Task 6: SPA embed and serving with the EC-15 cache contract

**Files:**
- Create: `web/embed.go`, `internal/httpapi/spa.go`
- Modify: `internal/httpapi/router.go` (delete the Task 5 stub)
- Test: `internal/httpapi/spa_test.go`

**Interfaces:**
- Consumes: `RouterConfig.SPA fs.FS` (Task 5).
- Produces: `web.DistFS() (fs.FS, error)` — the embedded `dist/` subtree; `spaHandler(dist fs.FS) http.HandlerFunc` (package-private, wired via `NewRouter`).

- [ ] **Step 1: Write `web/embed.go`**

```go
// Package web embeds the built admin SPA. The Go build fails when web/dist
// is missing — a binary with absent UI assets is unrepresentable
// (09-deployment.md §Build). Run vite build (make build does) first.
package web

import (
	"embed"
	"io/fs"
)

//go:embed all:dist
var dist embed.FS

// DistFS returns the dist/ subtree (index.html at its root).
func DistFS() (fs.FS, error) {
	return fs.Sub(dist, "dist")
}
```

- [ ] **Step 2: Write the failing SPA tests**

`internal/httpapi/spa_test.go`:

```go
package httpapi

import (
	"net/http"
	"net/http/httptest"
	"testing"
	"testing/fstest"
)

func spaFS() fstest.MapFS {
	return fstest.MapFS{
		"index.html":            &fstest.MapFile{Data: []byte("<html>app</html>")},
		"assets/index-abc123.js": &fstest.MapFile{Data: []byte("console.log(1)")},
	}
}

func TestSPA_CacheContract_EC15(t *testing.T) {
	h := spaHandler(spaFS())

	t.Run("hashed assets are immutable", func(t *testing.T) {
		rr := httptest.NewRecorder()
		h(rr, httptest.NewRequest(http.MethodGet, "/assets/index-abc123.js", nil))
		if rr.Code != 200 {
			t.Fatalf("status = %d", rr.Code)
		}
		if cc := rr.Header().Get("Cache-Control"); cc != "public, max-age=31536000, immutable" {
			t.Errorf("asset Cache-Control = %q", cc)
		}
	})

	t.Run("index.html is no-cache", func(t *testing.T) {
		rr := httptest.NewRecorder()
		h(rr, httptest.NewRequest(http.MethodGet, "/", nil))
		if cc := rr.Header().Get("Cache-Control"); cc != "no-cache" {
			t.Errorf("index Cache-Control = %q", cc)
		}
	})

	t.Run("SPA fallback serves index for client routes", func(t *testing.T) {
		rr := httptest.NewRecorder()
		h(rr, httptest.NewRequest(http.MethodGet, "/collections/posts", nil))
		if rr.Code != 200 || rr.Header().Get("Cache-Control") != "no-cache" {
			t.Errorf("fallback: status=%d cc=%q", rr.Code, rr.Header().Get("Cache-Control"))
		}
	})

	t.Run("missing asset under /assets is 404, not fallback", func(t *testing.T) {
		rr := httptest.NewRecorder()
		h(rr, httptest.NewRequest(http.MethodGet, "/assets/gone.js", nil))
		if rr.Code != 404 {
			t.Errorf("status = %d, want 404", rr.Code)
		}
	})
}
```

- [ ] **Step 3: Run to verify failure**

Run: `go test ./internal/httpapi/ -run TestSPA`
Expected: FAIL — stub serves 404 for everything.

- [ ] **Step 4: Write `internal/httpapi/spa.go` and delete the stub in `router.go`**

```go
package httpapi

import (
	"io/fs"
	"net/http"
	"strings"
)

// spaHandler serves the embedded admin SPA under the EC-15 cache-busting
// contract (09-deployment.md): content-hashed /assets/* are immutable for a
// year; index.html is revalidated on every navigation; every unknown
// non-API path falls back to index.html (client-side routing).
func spaHandler(dist fs.FS) http.HandlerFunc {
	fileServer := http.FileServer(http.FS(dist))
	return func(w http.ResponseWriter, r *http.Request) {
		p := strings.TrimPrefix(r.URL.Path, "/")
		if strings.HasPrefix(p, "assets/") {
			if _, err := fs.Stat(dist, p); err != nil {
				http.NotFound(w, r)
				return
			}
			w.Header().Set("Cache-Control", "public, max-age=31536000, immutable")
			fileServer.ServeHTTP(w, r)
			return
		}
		// Non-asset, non-API: serve index.html (SPA fallback).
		w.Header().Set("Cache-Control", "no-cache")
		index, err := fs.ReadFile(dist, "index.html")
		if err != nil {
			http.Error(w, "admin UI assets missing", http.StatusInternalServerError)
			return
		}
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		w.Write(index)
	}
}
```

In `router.go`, delete the temporary `spaHandler` stub added in Task 5.

- [ ] **Step 5: Run all package tests**

Run: `make unit` (builds `web/dist` first — `web/embed.go` now requires it to compile)
Expected: PASS across all packages.

- [ ] **Step 6: Commit**

```bash
git add web/embed.go internal/httpapi/
git commit -m "feat: embedded SPA serving with EC-15 cache-busting contract"
```

---

### Task 7: Instance lock with watchdog (BR-RUNTIME-8)

**Files:**
- Create: `internal/app/instancelock.go`
- Test: `internal/app/instancelock_integration_test.go`

**Interfaces:**
- Consumes: `testdb.URL` (Task 3).
- Produces: `app.acquireInstanceLock(ctx context.Context, databaseURL string, retryWindow, initialBackoff time.Duration, log *slog.Logger) (*instanceLock, error)`; `(*instanceLock).Lost() <-chan error` (closed-over error when the lock connection drops); `(*instanceLock).Release(ctx)`.

- [ ] **Step 1: Write the failing integration test**

`internal/app/instancelock_integration_test.go`:

```go
//go:build integration

package app

import (
	"context"
	"log/slog"
	"testing"
	"time"

	"github.com/codebabbler/golang-cms/internal/testdb"
	"github.com/jackc/pgx/v5"
)

func TestBR_RUNTIME_8_SecondProcessFailsAfterBoundedRetry(t *testing.T) {
	url := testdb.URL(t)
	ctx := context.Background()
	log := slog.Default()

	first, err := acquireInstanceLock(ctx, url, 5*time.Second, 50*time.Millisecond, log)
	if err != nil {
		t.Fatalf("first acquire: %v", err)
	}
	defer first.Release(ctx)

	start := time.Now()
	_, err = acquireInstanceLock(ctx, url, 1*time.Second, 50*time.Millisecond, log)
	if err == nil {
		t.Fatal("second acquire must fail while the first holds the lock")
	}
	if time.Since(start) < 1*time.Second {
		t.Errorf("second acquire gave up before the retry window elapsed")
	}
}

func TestBR_RUNTIME_8_WatchdogFiresWhenLockConnectionDies(t *testing.T) {
	url := testdb.URL(t)
	ctx := context.Background()

	lock, err := acquireInstanceLock(ctx, url, 5*time.Second, 50*time.Millisecond, slog.Default())
	if err != nil {
		t.Fatalf("acquire: %v", err)
	}
	defer lock.Release(context.Background())

	// Kill the lock connection server-side, identified by application_name.
	admin, err := pgx.Connect(ctx, url)
	if err != nil {
		t.Fatalf("admin connect: %v", err)
	}
	defer admin.Close(ctx)
	if _, err := admin.Exec(ctx,
		"SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE application_name = 'cms-instance-lock'",
	); err != nil {
		t.Fatalf("terminate: %v", err)
	}

	select {
	case err := <-lock.Lost():
		if err == nil {
			t.Error("Lost() must deliver a non-nil error")
		}
	case <-time.After(15 * time.Second):
		t.Fatal("watchdog did not fire within 15s of connection death")
	}
}

func TestInstanceLock_ReleaseAllowsReacquire(t *testing.T) {
	url := testdb.URL(t)
	ctx := context.Background()
	log := slog.Default()

	first, err := acquireInstanceLock(ctx, url, 5*time.Second, 50*time.Millisecond, log)
	if err != nil {
		t.Fatalf("first: %v", err)
	}
	first.Release(ctx)

	second, err := acquireInstanceLock(ctx, url, 2*time.Second, 50*time.Millisecond, log)
	if err != nil {
		t.Fatalf("reacquire after release: %v", err)
	}
	second.Release(ctx)
}
```

- [ ] **Step 2: Run to verify failure**

Run: `./scripts/testdb.sh go test -tags integration ./internal/app/ -run TestBR_RUNTIME_8 -v`
Expected: FAIL — `acquireInstanceLock` undefined.

- [ ] **Step 3: Write `internal/app/instancelock.go`**

```go
package app

import (
	"context"
	"fmt"
	"log/slog"
	"net"
	"time"

	"github.com/jackc/pgx/v5"
)

// instanceLockKey is the fixed instance advisory key (09-deployment.md
// §Startup). Session-scoped: the lock lives exactly as long as its
// dedicated connection (BR-RUNTIME-8).
const instanceLockKey = 0x636D7302

type instanceLock struct {
	conn *pgx.Conn
	lost chan error
	stop chan struct{}
}

// acquireInstanceLock takes pg_advisory_lock(instanceLockKey) on a dedicated
// connection with TCP keepalives, retrying with doubling backoff for up to
// retryWindow — riding out a crashed predecessor's lingering session — then
// fails with a clear error. Production values: 120s window (09 §Startup);
// tests inject shorter ones.
func acquireInstanceLock(ctx context.Context, databaseURL string, retryWindow, initialBackoff time.Duration, log *slog.Logger) (*instanceLock, error) {
	cfg, err := pgx.ParseConfig(databaseURL)
	if err != nil {
		return nil, fmt.Errorf("instance lock: parse config: %w", err)
	}
	cfg.RuntimeParams["application_name"] = "cms-instance-lock"
	cfg.DialFunc = (&net.Dialer{KeepAlive: 15 * time.Second}).DialContext

	conn, err := pgx.ConnectConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("instance lock: connect: %w", err)
	}

	deadline := time.Now().Add(retryWindow)
	backoff := initialBackoff
	for {
		var acquired bool
		if err := conn.QueryRow(ctx, "SELECT pg_try_advisory_lock($1)", instanceLockKey).Scan(&acquired); err != nil {
			conn.Close(context.WithoutCancel(ctx))
			return nil, fmt.Errorf("instance lock: try acquire: %w", err)
		}
		if acquired {
			break
		}
		if time.Now().After(deadline) {
			conn.Close(context.WithoutCancel(ctx))
			return nil, fmt.Errorf(
				"instance lock held by another process after %s of retries — one instance only (BR-RUNTIME-8)",
				retryWindow)
		}
		log.Warn("instance lock busy, retrying", "backoff", backoff.String())
		select {
		case <-ctx.Done():
			conn.Close(context.WithoutCancel(ctx))
			return nil, ctx.Err()
		case <-time.After(backoff):
		}
		if backoff < 10*time.Second {
			backoff *= 2
		}
	}

	l := &instanceLock{conn: conn, lost: make(chan error, 1), stop: make(chan struct{})}
	go l.watchdog()
	log.Info("instance lock acquired")
	return l, nil
}

// watchdog pings the lock connection; if it drops, serving is never
// permitted to continue (BR-RUNTIME-8) — the app exits on Lost().
func (l *instanceLock) watchdog() {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()
	for {
		select {
		case <-l.stop:
			return
		case <-ticker.C:
			ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
			err := l.conn.Ping(ctx)
			cancel()
			if err != nil {
				l.lost <- fmt.Errorf("instance-lock connection lost: %w", err)
				return
			}
		}
	}
}

func (l *instanceLock) Lost() <-chan error { return l.lost }

func (l *instanceLock) Release(ctx context.Context) {
	close(l.stop)
	// Closing the connection releases the session-scoped lock.
	l.conn.Close(context.WithoutCancel(ctx))
}
```

- [ ] **Step 4: Run to verify pass**

Run: `./scripts/testdb.sh go test -tags integration ./internal/app/ -v`
Expected: PASS (watchdog test takes ~5–10 s — the ping ticker interval).

- [ ] **Step 5: Commit**

```bash
git add internal/app/instancelock.go internal/app/instancelock_integration_test.go
git commit -m "feat: process-lifetime instance lock with keepalive watchdog (BR-RUNTIME-8)"
```

---

### Task 8: app.Run, cmd/cms, drain — the binary boots

**Files:**
- Create: `internal/app/app.go`, `cmd/cms/main.go`
- Modify: `Makefile` (add `build`)
- Test: `internal/app/run_integration_test.go`, `internal/app/drain_test.go`

**Interfaces:**
- Consumes: everything — `LoadConfig` (T2), `store.Migrate` (T3), `audit` (T4), `httpapi.NewRouter`/`NewHealthState` (T5), `web.DistFS` (T6), `acquireInstanceLock` (T7).
- Produces: `app.Run(ctx context.Context, cfg *Config, log *slog.Logger, opts ...Option) error`; `app.Option` values `WithLockRetry(window, initialBackoff time.Duration)`, `WithDrainTimeout(d time.Duration)`, `WithOnReady(f func(addr net.Addr))`.

- [ ] **Step 1: Write the failing tests**

`internal/app/run_integration_test.go`:

```go
//go:build integration

package app

import (
	"context"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"testing"
	"time"

	"github.com/codebabbler/golang-cms/internal/testdb"
)

func bootConfig(url string) *Config {
	return &Config{
		DatabaseURL:        url,
		MasterSecret:       "0123456789abcdef0123456789abcdef",
		Port:               0, // ephemeral
		LogLevel:           slog.LevelInfo,
		TrashRetentionDays: 30,
		RevisionLimit:      50,
	}
}

// boot starts Run and returns the base URL, a cancel func, and the Run error channel.
func boot(t *testing.T, url string) (string, context.CancelFunc, <-chan error) {
	t.Helper()
	ctx, cancel := context.WithCancel(context.Background())
	ready := make(chan net.Addr, 1)
	errCh := make(chan error, 1)
	go func() {
		errCh <- Run(ctx, bootConfig(url), slog.Default(),
			WithLockRetry(5*time.Second, 50*time.Millisecond),
			WithOnReady(func(a net.Addr) { ready <- a }),
		)
	}()
	select {
	case addr := <-ready:
		// Dial loopback explicitly — addr.String() may be "[::]:port".
		return fmt.Sprintf("http://127.0.0.1:%d", addr.(*net.TCPAddr).Port), cancel, errCh
	case err := <-errCh:
		cancel()
		t.Fatalf("Run exited before ready: %v", err)
		return "", nil, nil
	case <-time.After(30 * time.Second):
		cancel()
		t.Fatal("Run not ready within 30s")
		return "", nil, nil
	}
}

func TestBR_RUNTIME_3_StartupOrderBootsMigratesAndServes(t *testing.T) {
	base, cancel, errCh := boot(t, testdb.URL(t))
	defer cancel()

	for _, path := range []string{"/healthz", "/readyz"} {
		resp, err := http.Get(base + path)
		if err != nil {
			t.Fatalf("GET %s: %v", path, err)
		}
		resp.Body.Close()
		if resp.StatusCode != 200 {
			t.Errorf("%s = %d, want 200", path, resp.StatusCode)
		}
	}

	// Unknown /api route answers through WriteError (BR-API-3).
	resp, err := http.Get(base + "/api/nope")
	if err != nil {
		t.Fatalf("GET /api/nope: %v", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != 404 {
		t.Errorf("/api/nope = %d, want 404", resp.StatusCode)
	}
	if ct := resp.Header.Get("Content-Type"); ct != "application/json; charset=utf-8" {
		t.Errorf("error Content-Type = %q — must come from WriteError", ct)
	}

	cancel()
	select {
	case err := <-errCh:
		if err != nil {
			t.Errorf("clean shutdown must return nil, got %v", err)
		}
	case <-time.After(20 * time.Second):
		t.Fatal("Run did not return after cancel")
	}
}

func TestBR_RUNTIME_3_SecondInstanceFailsStartup(t *testing.T) {
	url := testdb.URL(t)
	_, cancel, _ := boot(t, url)
	defer cancel()

	// Timeout guard: if the second instance wrongly acquires the lock it
	// would serve forever; the deadline turns that bug into a test failure.
	ctx, cancelSecond := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancelSecond()
	err := Run(ctx, bootConfig(url), slog.Default(),
		WithLockRetry(1*time.Second, 50*time.Millisecond))
	if err == nil {
		t.Fatal("second instance must fail startup (BR-RUNTIME-8 / EC-16)")
	}
}
```

`internal/app/drain_test.go` (unit — drain logic tested against a local server):

```go
package app

import (
	"context"
	"io"
	"log/slog"
	"net/http"
	"net/http/httptest"
	"sync/atomic"
	"testing"
	"time"

	"github.com/codebabbler/golang-cms/internal/httpapi"
)

func TestBR_RUNTIME_6_DrainCompletesInflightWithinWindow(t *testing.T) {
	health := httpapi.NewHealthState()
	var inflight atomic.Int64
	slow := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inflight.Add(1)
		defer inflight.Add(-1)
		time.Sleep(300 * time.Millisecond)
		w.Write([]byte("done"))
	})
	srv := httptest.NewServer(slow)
	defer srv.Close()

	got := make(chan string, 1)
	go func() {
		resp, err := http.Get(srv.URL)
		if err != nil {
			got <- "error: " + err.Error()
			return
		}
		defer resp.Body.Close()
		b, _ := io.ReadAll(resp.Body)
		got <- string(b)
	}()
	time.Sleep(50 * time.Millisecond) // request is in flight

	err := drain(context.Background(), srv.Config, health, &inflight, 2*time.Second, slog.Default())
	if err != nil {
		t.Fatalf("drain within window must return nil, got %v", err)
	}
	if !health.Draining() {
		t.Error("drain must flip HealthState first (EC-14 step 1)")
	}
	if body := <-got; body != "done" {
		t.Errorf("in-flight request dropped: %q", body)
	}
}

func TestBR_RUNTIME_6_DrainForceClosesStragglersAndErrors(t *testing.T) {
	health := httpapi.NewHealthState()
	var inflight atomic.Int64
	stuck := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inflight.Add(1)
		defer inflight.Add(-1)
		select {
		case <-r.Context().Done():
		case <-time.After(30 * time.Second):
		}
	})
	srv := httptest.NewServer(stuck)
	defer srv.Close()

	go http.Get(srv.URL)
	time.Sleep(50 * time.Millisecond)

	err := drain(context.Background(), srv.Config, health, &inflight, 200*time.Millisecond, slog.Default())
	if err == nil {
		t.Fatal("drain exceeding the window must force-close and return an error (exit 1)")
	}
}
```

- [ ] **Step 2: Run to verify failure**

Run: `go test ./internal/app/ -run TestBR_RUNTIME_6`
Expected: FAIL — `drain` and `Run` undefined.

- [ ] **Step 3: Write `internal/app/app.go`**

```go
// Package app is the composition root: startup order (BR-RUNTIME-3),
// shutdown drain (EC-14), and wiring. Nothing imports httpapi except this
// package and cmd (10-project-structure.md §Package Rules).
package app

import (
	"context"
	"errors"
	"fmt"
	"log/slog"
	"net"
	"net/http"
	"sync/atomic"
	"time"

	"github.com/codebabbler/golang-cms/internal/httpapi"
	"github.com/codebabbler/golang-cms/internal/store"
	"github.com/codebabbler/golang-cms/web"
	"github.com/jackc/pgx/v5/pgxpool"
)

// Compiled-in constants (09-deployment.md §Timeouts, §Startup).
const (
	readHeaderTimeout  = 5 * time.Second
	readTimeout        = 30 * time.Second
	writeTimeout       = 30 * time.Second
	idleTimeout        = 120 * time.Second
	poolMaxConns       = 10
	defaultDrainWindow = 15 * time.Second
	defaultLockWindow  = 120 * time.Second
	defaultLockBackoff = 1 * time.Second
)

type options struct {
	lockWindow  time.Duration
	lockBackoff time.Duration
	drainWindow time.Duration
	onReady     func(net.Addr)
}

type Option func(*options)

// WithLockRetry overrides the instance-lock retry window and initial backoff
// (integration tests; production uses the 09 §Startup 120s window).
func WithLockRetry(window, initialBackoff time.Duration) Option {
	return func(o *options) { o.lockWindow, o.lockBackoff = window, initialBackoff }
}

func WithDrainTimeout(d time.Duration) Option {
	return func(o *options) { o.drainWindow = d }
}

// WithOnReady reports the bound listener address once /readyz starts
// returning 200 (startup step 6).
func WithOnReady(f func(net.Addr)) Option {
	return func(o *options) { o.onReady = f }
}

// Run executes the startup order of BR-RUNTIME-3 and blocks until ctx is
// cancelled (clean drain, returns nil), a fatal serve error occurs, or the
// instance lock is lost (BR-RUNTIME-8 — returns non-nil; main exits 1).
func Run(ctx context.Context, cfg *Config, log *slog.Logger, opts ...Option) error {
	o := &options{
		lockWindow:  defaultLockWindow,
		lockBackoff: defaultLockBackoff,
		drainWindow: defaultDrainWindow,
	}
	for _, opt := range opts {
		opt(o)
	}

	// Step 1: config is already validated (LoadConfig); open the pool.
	poolCfg, err := pgxpool.ParseConfig(cfg.DatabaseURL)
	if err != nil {
		return fmt.Errorf("startup: parse DATABASE_URL: %w", err)
	}
	poolCfg.MaxConns = poolMaxConns
	pool, err := pgxpool.NewWithConfig(ctx, poolCfg)
	if err != nil {
		return fmt.Errorf("startup: open pool: %w", err)
	}
	defer pool.Close()

	// Step 2: embedded migrations under the migration advisory lock.
	// Failure aborts startup — fail closed (N-11).
	if _, err := store.Migrate(ctx, pool, log); err != nil {
		return fmt.Errorf("startup: migrations: %w", err)
	}

	// Step 3: process-lifetime instance lock (BR-RUNTIME-8).
	lock, err := acquireInstanceLock(ctx, cfg.DatabaseURL, o.lockWindow, o.lockBackoff, log)
	if err != nil {
		return fmt.Errorf("startup: %w", err)
	}
	defer lock.Release(context.WithoutCancel(ctx))

	// Step 4 seam: schema-cache load (Phase 3 fills this; failure will
	// abort startup per N-11).
	// Step 5 seam: setup/recovery token generation (Phase 2; BR-AUTH-11/12).

	// Step 6: open the listener.
	dist, err := web.DistFS()
	if err != nil {
		return fmt.Errorf("startup: embedded UI: %w", err)
	}
	health := httpapi.NewHealthState()
	router := httpapi.NewRouter(httpapi.RouterConfig{
		Log:    log,
		DB:     pool,
		Health: health,
		SPA:    dist,
	})

	var inflight atomic.Int64
	counted := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inflight.Add(1)
		defer inflight.Add(-1)
		router.ServeHTTP(w, r)
	})

	server := &http.Server{
		Handler:           counted,
		ReadHeaderTimeout: readHeaderTimeout,
		ReadTimeout:       readTimeout,
		WriteTimeout:      writeTimeout,
		IdleTimeout:       idleTimeout,
	}

	ln, err := net.Listen("tcp", fmt.Sprintf(":%d", cfg.Port))
	if err != nil {
		return fmt.Errorf("startup: listen on port %d: %w", cfg.Port, err)
	}
	log.Info("listening", "addr", ln.Addr().String(), "media_enabled", cfg.MediaEnabled())
	if o.onReady != nil {
		o.onReady(ln.Addr())
	}

	serveErr := make(chan error, 1)
	go func() { serveErr <- server.Serve(ln) }()

	select {
	case <-ctx.Done():
		return drain(context.WithoutCancel(ctx), server, health, &inflight, o.drainWindow, log)
	case err := <-lock.Lost():
		// Serving without the lock is never permitted (BR-RUNTIME-8):
		// close immediately, no drain, exit non-zero.
		log.Error("instance lock lost, exiting", "err", err)
		server.Close()
		return err
	case err := <-serveErr:
		if errors.Is(err, http.ErrServerClosed) {
			return nil
		}
		return fmt.Errorf("serve: %w", err)
	}
}

// drain implements EC-14: flip health to 503 → (job tickers stop here from
// P8) → http.Server.Shutdown within the window → exit 0; stragglers are
// force-closed, logged and counted, and the process exits 1 (BR-RUNTIME-6).
func drain(ctx context.Context, server *http.Server, health *httpapi.HealthState, inflight *atomic.Int64, window time.Duration, log *slog.Logger) error {
	health.SetDraining(true)
	log.Info("drain started", "window", window.String())

	shutdownCtx, cancel := context.WithTimeout(ctx, window)
	defer cancel()
	err := server.Shutdown(shutdownCtx)
	if err == nil {
		log.Info("drain complete")
		return nil
	}
	count := inflight.Load()
	log.Error("drain window exceeded, force-closing stragglers",
		"force_closed", count, "window", window.String())
	server.Close()
	return fmt.Errorf("drain force-closed %d in-flight requests (BR-RUNTIME-6)", count)
}
```

- [ ] **Step 4: Write `cmd/cms/main.go`**

```go
// The cms binary. Flag-free: env config only (10-project-structure.md).
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"os/signal"
	"syscall"

	"github.com/codebabbler/golang-cms/internal/app"
)

func main() {
	cfg, err := app.LoadConfig(os.LookupEnv)
	if err != nil {
		// Startup failure lists every missing variable at once
		// (09-deployment.md §Configuration).
		fmt.Fprintln(os.Stderr, err.Error())
		os.Exit(1)
	}

	log := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{Level: cfg.LogLevel}))
	slog.SetDefault(log)

	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGTERM, syscall.SIGINT)
	defer stop()

	if err := app.Run(ctx, cfg, log); err != nil {
		log.Error("exited", "err", err)
		os.Exit(1)
	}
}
```

- [ ] **Step 5: Finalize `make build`**

Add to `Makefile`:

```make
build: web/dist/index.html
	go build -o bin/cms ./cmd/cms
```

- [ ] **Step 6: Run everything**

Run: `make build && make test`
Expected: build produces `bin/cms`; all unit + integration tests PASS (drain, boot, second-instance, watchdog, migrate, config, httpapi, audit, SPA).

- [ ] **Step 7: Manual smoke (acceptance criteria 2, 3, 6)**

```bash
./scripts/testdb.sh bash -c '
  DATABASE_URL="$TEST_DATABASE_URL" CMS_MASTER_SECRET=0123456789abcdef0123456789abcdef ./bin/cms &
  pid=$!
  sleep 2
  curl -fsS -o /dev/null -w "healthz %{http_code}\n" localhost:8080/healthz
  curl -fsS -o /dev/null -w "readyz %{http_code}\n"  localhost:8080/readyz
  kill -TERM $pid
  wait $pid && echo "exit 0 on SIGTERM"
'
./bin/cms; echo "exit=$? (want 1, listing DATABASE_URL and CMS_MASTER_SECRET)"
```

Expected: both 200s, `exit 0 on SIGTERM`, then a non-zero exit naming both required variables.

- [ ] **Step 8: Commit**

```bash
git add internal/app/app.go internal/app/run_integration_test.go internal/app/drain_test.go cmd/ Makefile
git commit -m "feat: app.Run startup order, drain, and cms binary — the skeleton walks"
```

---

### Task 9: Trace gate, waiver file, depguard, doc amendment, acceptance sweep

**Files:**
- Create: `scripts/trace.sh`, `docs/trace-waivers.txt`, `.golangci.yml`
- Modify: `Makefile` (add `trace`, `lint`), `docs/architecture/07-data-model.md` (cms_migrations row + version bump)

**Interfaces:**
- Consumes: the BR-named tests written in Tasks 2–8.
- Produces: `make trace` green with the waiver discipline of D-P1-2.

- [ ] **Step 1: Write `scripts/trace.sh`** (make it executable)

```bash
#!/usr/bin/env bash
# BR coverage gate (N-10, BUSINESS_RULES.md §Rule-to-Code Traceability).
# Fails on any non-structural BR with neither a test nor a waiver, and on
# any waived BR that now has a test (stale waiver — shrink the file, D-P1-2).
set -euo pipefail
manual=docs/BUSINESS_RULES.md
waivers=docs/trace-waivers.txt
fail=0

tested() {
  # Trailing boundary so BR-AUTH-1 never matches a BR-AUTH-11 test.
  local id=$1 underscored=${1//-/_}
  grep -rlqE "(${id}|${underscored})([^0-9]|\$)" --include='*_test.go' . 2>/dev/null
}

for id in $(grep -oE 'BR-[A-Z]+-[0-9]+' "$manual" | sort -u); do
  # Structural rules are exempt: enforcement is the absence of an
  # alternative code path, verified by review.
  if grep -F "**${id}.**" "$manual" | grep -q '\[structural\]'; then
    continue
  fi
  if grep -qE "^${id}[[:space:]]" "$waivers"; then
    if tested "$id"; then
      echo "TRACE STALE: ${id} is waived but has a test — remove it from ${waivers}"
      fail=1
    fi
    continue
  fi
  if ! tested "$id"; then
    echo "TRACE FAIL: ${id} has no test and no waiver"
    fail=1
  fi
done

if [ "$fail" -eq 0 ]; then
  remaining=$(grep -cE '^BR-' "$waivers" || true)
  echo "trace OK (${remaining} BR(s) waived pending later phases; must reach 0 by the V1 gate)"
fi
exit $fail
```

```bash
chmod +x scripts/trace.sh
```

- [ ] **Step 2: Write `docs/trace-waivers.txt`**

Every non-structural BR not covered by a P1 test, annotated with its owning phase per the phasing spec. P1 covers BR-RUNTIME-3/6/8, BR-API-3, BR-AUDIT-1/2, BR-MEDIA-6, BR-RBAC-1; structural (exempt, not listed): BR-RUNTIME-1/2, BR-MEDIA-1/4.

```text
# BR-IDs pending a later phase (D-P1-2, phasing spec 2026-07-11).
# Format: <BR-ID> <owning phase>. A phase's plan shrinks this file; it must
# be EMPTY at the V1 delivery gate. make trace fails on stale entries.
BR-AUTH-1 P2
BR-AUTH-2 P2
BR-AUTH-3 P2
BR-AUTH-4 P2
BR-AUTH-5 P2
BR-AUTH-6 P2
BR-AUTH-11 P2
BR-AUTH-12 P2
BR-RUNTIME-4 P2
BR-SCHEMA-1 P3
BR-SCHEMA-2 P3
BR-SCHEMA-4 P3
BR-SCHEMA-5 P3
BR-SCHEMA-6 P3
BR-SCHEMA-7 P3
BR-SCHEMA-8 P3
BR-RUNTIME-7 P3
BR-SCHEMA-3 P4
BR-LIFE-1 P4
BR-LIFE-2 P4
BR-LIFE-3 P4
BR-LIFE-4 P4
BR-LIFE-5 P4
BR-LIFE-7 P4
BR-RBAC-5 P4
BR-API-1 P4
BR-AUTH-7 P5
BR-AUTH-8 P5
BR-AUTH-9 P5
BR-AUTH-10 P5
BR-AUTH-13 P5
BR-AUTH-14 P5
BR-API-2 P6
BR-API-4 P6
BR-API-5 P6
BR-API-6 P6
BR-API-7 P6
BR-RBAC-2 P6
BR-RBAC-3 P6
BR-RBAC-4 P6
BR-RBAC-6 P6
BR-RBAC-7 P6
BR-LIFE-6 P6
BR-MEDIA-3 P7
BR-MEDIA-5 P7
BR-LIFE-8 P8
BR-MEDIA-2 P8
BR-RUNTIME-5 P8
BR-AUDIT-3 V2
BR-LIFE-9 V2
```

- [ ] **Step 3: Write `.golangci.yml`** — the 10-project-structure §Package Rules seams, configured now so they bind from each package's first commit

```yaml
version: "2"
linters:
  enable:
    - depguard
  settings:
    depguard:
      rules:
        no-httpapi-imports:
          files:
            - "!**/internal/app/**"
            - "!**/cmd/**"
            - "!**/internal/httpapi/**"
          deny:
            - pkg: "github.com/codebabbler/golang-cms/internal/httpapi"
              desc: "nothing imports httpapi except app and cmd (10-project-structure)"
        squirrel-only-in-query:
          files:
            - "!**/internal/query/**"
          deny:
            - pkg: "github.com/Masterminds/squirrel"
              desc: "squirrel imports only in internal/query (BR-SCHEMA-3)"
        storage-sdk-only-in-media:
          files:
            - "!**/internal/media/**"
          deny:
            - pkg: "github.com/aws/aws-sdk-go-v2"
              desc: "the storage SDK imports only in internal/media (10-project-structure)"
```

- [ ] **Step 4: Add `trace` and `lint` to the Makefile**

```make
trace:
	./scripts/trace.sh

lint:
	golangci-lint run
```

(If `golangci-lint` is not installed: `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest` — dev-time tool, not a runtime dependency.)

- [ ] **Step 5: Run the gate**

Run: `make trace`
Expected: `trace OK (50 BR(s) waived pending later phases; must reach 0 by the V1 gate)` — exit 0. If any TRACE FAIL/STALE line prints, fix the waiver file or the test naming before proceeding.

Run: `make lint`
Expected: clean.

- [ ] **Step 6: Amend `docs/architecture/07-data-model.md`**

In §System Tables (V1), after the `cms_media_deletions` row, add:

```markdown
| `cms_migrations` | `version` (PK), `name`, `applied_at` | Migration ledger, written only by the embedded startup runner (BR-RUNTIME-3); append-only, never referenced elsewhere. |
```

Bump the header line from `**Version:** 1.3` to `**Version:** 1.4` (same `Last Updated` date format, today's date).

- [ ] **Step 7: Full acceptance sweep — all 11 criteria from the phasing spec**

```bash
# 1. build works; fails without web/dist
make build && ls bin/cms
mv web/dist /tmp/dist-backup && (go build ./... 2>&1 | grep -q "pattern all:dist" && echo "criterion 1b OK: build fails without dist"); mv /tmp/dist-backup web/dist
# 2+6. boot, health, tables, SIGTERM — covered by Task 8 step 7 and make test
# 3. missing env lists both
env -i ./bin/cms 2>&1 | grep -c "DATABASE_URL\|CMS_MASTER_SECRET"   # expect 2 problem lines
# 4. partial S3
env -i DATABASE_URL=x CMS_MASTER_SECRET=y S3_BUCKET=b ./bin/cms 2>&1 | grep "S3_ENDPOINT"
# 5. second instance + watchdog — integration tests (make test)
# 7. envelope + closed registry — unit tests (make unit)
# 8. cache headers — spa unit tests + manual: curl -sI localhost:8080/ | grep -i cache-control
# 9. gates
make test && make trace && make generate && git diff --exit-code internal/store/
# 10. depguard
make lint
# 11. doc row present
grep -n "cms_migrations" docs/architecture/07-data-model.md
```

Expected: every line succeeds. Record any deviation instead of forcing it green.

- [ ] **Step 8: Commit**

```bash
git add scripts/trace.sh docs/trace-waivers.txt .golangci.yml Makefile docs/architecture/07-data-model.md
git commit -m "feat: BR trace gate with phase waivers, depguard seams, cms_migrations doc row"
```

---

## Plan Self-Review Notes

- **Spec coverage:** criteria 1–11 map to Tasks 1 (shell/build), 2 (config, criterion 3/4), 3 (migration/ledger, criterion 2), 5 (registry, criterion 7), 6 (cache headers, criterion 8), 7–8 (lock/watchdog/drain/boot, criteria 2/5/6), 9 (trace/depguard/doc, criteria 9/10/11). Startup seams for steps 4/5 live in `app.Run` comments. The audit interface deviation (ctx param) is documented in Task 4's interface block.
- **Sequencing:** audit (T4) precedes httpapi (T5) because RequestID middleware writes audit's context key; web embed (T6) precedes app.Run (T8) because Run loads `web.DistFS()`.
- **Known intentional deviations to flag in review:** `Emit(ctx, Event)` vs 02's schematic `Emit(e Event)` (request-id correlation requires ctx); `cms_reset_tokens.user_id` carries no FK (polymorphic — cascade is application-level in P2).
