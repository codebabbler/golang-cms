# V3-P1 — Commerce Foundation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** All V3 configuration parses from day one with graceful 503-absence modes, collections become commerce-designatable via `commerce_config`, and the trace harness enters V3 mode.

**Architecture:** Two new atomic env groups follow the `S3_*`/media-less precedent exactly: `app.Config` gains `Commerce *CommerceEnv` and `Video *VideoEnv` (nil = mode off), with `RequireCommerce`/`RequireVideo` route guards returning 503 `unavailable`. `commerce_config` is a JSONB column on `cms_collections` (migration 0009) managed by a new `schema.Engine` op with validation that protects the designated price field from destructive schema changes. No V3-tagged BRs exist, so the trace flip is pure hygiene and the waiver stays empty.

**Tech Stack:** Go 1.25, pgx/v5, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v3-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code; docs win over drifted plan detail.
- Money is integer minor units (`BIGINT` cents); the commerce price field must be a `number` with scale 0; currency pinned per collection, ISO 4217 lowercase (spec B1).
- New env vars ONLY as the two atomic groups defined here (D-V3-7); partial group → startup failure listing missing names.
- DDL only through `schema.Engine`'s whitelist or migration `0009`; errors only via `httpapi.WriteError`; audit + vocabulary growth (`schema.commerce_config.update`).
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V3-2 — run before Task 1)

- [ ] Confirm: `app.LoadConfig(lookup func(string)(string,bool)) (*Config, error)`, `app.Config{…S3 *S3Config…}`, `(*Config).MediaEnabled()`, `app.ConfigError{Problems []string}` (V1-P1 T2) — the new groups clone the S3 pattern.
- [ ] Confirm the media-less 503 guard's shape and `details` key (V1-P7: media-less → 503 `unavailable`) — commerce/video guards must return the identical envelope with `{"reason":"commerce_disabled"}` / `{"reason":"video_disabled"}`.
- [ ] Confirm `schema.Engine.Apply` op dispatch, the `access_rules`/`search_config` validation seams (V1-P3, V2-P5) — `commerce_config` is the third config surface on `cms_collections` and follows `search_config`'s op pattern (V2-P5 T2).
- [ ] Confirm `scripts/trace.sh` still carries the `\(V3\)` exemption clause (V2-P1 T6 left it); confirm `docs/trace-waivers.txt` is empty.
- [ ] Confirm migrations through `0008` (V2 complete); `0009` next — renumber if drifted.

## File Structure

```
internal/app/config.go                  CommerceEnv, VideoEnv groups (modify)
internal/httpapi/mode_guards.go         RequireCommerce, RequireVideo
internal/store/migrations/0009_commerce_config.sql
internal/schema/commerce.go             CommerceConfig + validation + OpSetCommerceConfig
web/src/routes/CollectionEdit.svelte    Commerce section (modify)
web/e2e/v3p1-commerce-config.spec.ts
scripts/trace.sh                        V3 exemption dropped (modify)
docs/architecture/09-deployment.md      env table rows (amend)
docs/architecture/07-data-model.md      commerce_config column (amend)
```

---

### Task 1: Env groups + 503 mode guards + 09 amendment

**Files:**
- Modify: `internal/app/config.go`, `docs/architecture/09-deployment.md`
- Create: `internal/httpapi/mode_guards.go`
- Test: extend `internal/app/config_test.go`; `internal/httpapi/mode_guards_test.go`

**Interfaces:**
- Produces:
  - `app.CommerceEnv{SecretKey, WebhookSecret string}` from `STRIPE_SECRET_KEY` + `STRIPE_WEBHOOK_SECRET`; `app.VideoEnv{KeyID, KeyPEM, CustomerCode string}` from `CF_STREAM_KEY_ID` + `CF_STREAM_KEY_PEM` (base64-encoded PEM — decoded and PEM-parse-validated at load; invalid → ConfigError) + `CF_STREAM_CUSTOMER_CODE`. `Config.Commerce *CommerceEnv`, `Config.Video *VideoEnv`; `(*Config).CommerceEnabled() bool`, `(*Config).VideoEnabled() bool`. Group atomicity: all-or-nothing per group; a partial group adds one Problem line per missing/invalid variable naming it exactly (BR-MEDIA-6 pattern).
  - `httpapi.RequireCommerce(enabled bool) func(http.Handler) http.Handler` / `RequireVideo(enabled bool)` — disabled → `WriteError(w, r, unavailable, "commerce is not configured", map[string]any{"reason":"commerce_disabled"})` (503; video variant mirrors). Consumed by V3-P2/P3 (commerce) and V3-P4 (video).

- [ ] **Step 1: Failing tests** — config table tests: both groups absent → nil pointers, `CommerceEnabled()==false`; full Stripe pair → populated; `STRIPE_SECRET_KEY` alone → ConfigError with one line naming `STRIPE_WEBHOOK_SECRET`; CF trio minus `CF_STREAM_CUSTOMER_CODE` → line naming it; non-base64 or non-PEM `CF_STREAM_KEY_PEM` → line naming it with reason. Guard tests: disabled guard → 503, code `unavailable`, `details.reason` exact; enabled guard passes through.
- [ ] **Step 2:** FAIL. **Step 3:** implement (clone the `S3Config` load block twice; guards ≈ 15 lines). **Step 4:** PASS.
- [ ] **Step 5: Amend 09** — env table gains five rows (names, formats, "optional — group-atomic; absent group disables commerce/video mode with 503 on those routes"); startup section notes the PEM decode failure aborts.
- [ ] **Step 6: Commit** — `git commit -m "feat: STRIPE_* and CF_STREAM_* atomic env groups with 503 mode guards (D-V3-7)"`

---

### Task 2: Migration 0009 + `CommerceConfig` engine op + 07 amendment

**Files:**
- Create: `internal/store/migrations/0009_commerce_config.sql` (one line: `ALTER TABLE cms_collections ADD COLUMN commerce_config JSONB;`), `internal/schema/commerce.go`
- Modify: `internal/schema/engine.go` (op dispatch + price-field protection), snapshot types (carry the config), admin collections handler (expose the op), `internal/audit/actions.go` (+`schema.commerce_config.update`), `docs/architecture/07-data-model.md`
- Test: `internal/schema/commerce_test.go`

**Interfaces:**
- Produces:
  - `schema.CommerceConfig{Enabled bool; PriceField string; Currency string}` (JSON `{"enabled","priceField","currency"}`).
  - `schema.ValidateCommerceConfig(col Collection, cfg CommerceConfig) error` — `PriceField` names an existing `number` field with scale 0 (nil-or-zero `Scale` — re-validate how V1 stores "scale 0" vs "bare NUMERIC": **bare `NUMERIC` (no declared scale) is NOT integer-safe and is rejected**; the error text says "price field must be a number with declared scale 0"); `Currency` ∈ the closed lowercase list `usd, eur, gbp, jpy, aud, cad, chf, sek, nok, dkk, inr, npr, sgd, nzd` (grow-only, code-reviewed additions).
  - `schema.OpSetCommerceConfig{CollectionID uuid.UUID; Config CommerceConfig}` — standard `Apply` sequence (advisory lock, metadata write, snapshot swap, audit `schema.commerce_config.update`). Clearing = `Enabled:false` config persisted as NULL.
  - **Price-field protection:** field drop and type-change ops targeting a field named as an enabled collection's `PriceField` → rejected with the engine's inbound-reference error pattern naming `commerce_config` (remediation: disable commerce or repoint the price field first).
  - Snapshot: `Collection.Commerce *CommerceConfig` — consumed by V3-P2 (cart validation), V3-P3 (checkout pricing), V3-P5 (product-block pickers).
  - Route: `PUT /api/admin/collections/{id}/commerce-config` — `RequireSession`+`RequireCSRF`+`RequireRole(admin)` (matches the search-config floor).

- [ ] **Step 1: Failing tests** — set valid config on a collection with `price number(scale 0)` → persisted, snapshot carries it, audit event present; `priceField` naming a `text` field → 422 naming it; bare-NUMERIC price field → 422 with the declared-scale message; unknown currency `btc` → 422; drop the designated price field → rejected naming `commerce_config`; type-change it → rejected; disable commerce → the same drop now succeeds.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS.
- [ ] **Step 5: Amend 07** — `cms_collections` row gains `commerce_config JSONB (V3)` → written as live present-tense: "names the price field and pins the currency for commerce-enabled collections (D-V3-5)".
- [ ] **Step 6: Commit** — `git commit -m "feat: commerce_config engine op with price-field protection — migration 0009 (D-V3-5)"`

---

### Task 3: Admin UI commerce section + e2e

**Files:**
- Modify: `web/src/routes/CollectionEdit.svelte`
- Create: `web/e2e/v3p1-commerce-config.spec.ts`

- [ ] **Step 1: Failing e2e** — create a collection with `title(text)` + `price(number, scale 0)`; open the new "Commerce" section: toggle enable, price-field select lists only eligible fields (`price`, not `title`), currency select; save → section shows enabled state after reload; attempt to delete the `price` field → the engine's 422 surfaces naming `commerce_config`; disable commerce → delete succeeds.
- [ ] **Step 2: Implement** — collapsible "Commerce" section (enable toggle; when enabled: price-field select filtered client-side to `number` fields with scale 0, currency select over the closed list fetched from a small `GET /api/admin/commerce/currencies` constant endpoint — or inline the list in the bundle; **inline it**: the list is code-constant either way and an endpoint is YAGNI); save via the Task 2 route; `FieldErrors` surfaces 422s.
- [ ] **Step 3:** e2e PASS. **Step 4: Commit** — `git commit -m "feat: commerce section in collection editor (D-V3-5)"`

---

### Task 4: V3 harness flip + acceptance sweep

**Files:**
- Modify: `scripts/trace.sh`

- [ ] **Step 1: Flip** — delete the `\(V3\)` exemption clause (the `if grep -qE "\*\*${id} \(V3\)\.\*\*" …` block V2-P1 left behind) and its comment; `grep -c '(V3)' docs/BUSINESS_RULES.md` → confirm the tag appears nowhere (dormant flip — zero V3-tagged rules exist; new V3 BRs arrive untagged-with-tests in P3/P4).
- [ ] **Step 2:** `make trace` → green; `grep -c '^BR-' docs/trace-waivers.txt` → `0`.
- [ ] **Step 3: Sweep** — `make test` green; full e2e suite (V1 + v2p1–p8 + v3p1) green; `make bench` unchanged; **503-mode proof**: boot the binary with no V3 env → `GET /api/v1/cart` (route exists from V3-P2 — for THIS phase assert the guards directly via the Task 1 unit tests and a temporary probe route in the integration harness; re-validate: if no guarded route exists yet, the guard unit tests are the phase's proof and the boot probe moves to V3-P2's sweep); config round-trip: boot WITH both groups → `CommerceEnabled()` and `VideoEnabled()` true, startup log lines name both modes active (no secrets logged — assert against the V1 secret-log suite).
- [ ] **Step 4: Commit** — `git commit -m "chore: trace harness V3 flip; commerce foundation sweep"`

## Self-Review Notes (execution-time attention)

- **Bare-`NUMERIC` rejection** is stricter than "scale 0 by default": Postgres bare NUMERIC accepts fractional values, so it cannot guarantee integer minor units — the declared-scale-0 requirement is load-bearing for BR-COM-1 (V3-P3), not pedantry.
- **`CF_STREAM_KEY_PEM` base64-wrapped** because raw multi-line PEM in env vars breaks most process managers; decode-then-parse at load fails fast (N-11 posture).
- **Currency list is code-constant and inlined in the UI bundle** — flagged: if reviewers prefer an endpoint, it's a 10-line addition, but two sources of the same constant is the actual risk; keep one Go constant + one generated JS constant emitted into the bundle at build if drift worries the reviewer (execution-time choice).
- **Price-field protection uses the inbound-reference error pattern** rather than a new sentinel — one less error shape; re-validate the exact V1 sentinel to reuse.
