# V3-P4 — Video Paywalls Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Paywalled video fields play only through short-lived signed tokens issued to entitled end users (UAC-3.2, F-30, new BR-VID-1); the binary signs tokens and never touches video bytes (O-8).

**Architecture:** A new `video` field type stores the provider UID as `TEXT` with a `paywalled` config flag; serialization always renders `{uid, paywalled}` — never a URL. `internal/video` defines `Provider{PlaybackToken}` with one reference adapter: Cloudflare Stream RS256 JWTs (kid from `CF_STREAM_KEY_ID`, 10-minute exp). Entitlements (`UNIQUE(end_user_id, record_id)`) are written by V3-P3's `OrderPaidHook` plus an idempotent every-boot backfill, and checked by the token endpoint: anonymous-on-paywalled → **403** (UAC-3.2's explicit code).

**Tech Stack:** Go 1.25, pgx/v5, RS256 JWT (stdlib crypto — the V1-P5 JWT machinery's signing primitives), Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v3-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- **BR-VID-1 (added this phase):** paywalled playback exclusively via short-lived (≤ 15 min) signed tokens issued to entitled end users; the binary signs only and never proxies/stores video bytes (O-8); video field serialization never contains a playback URL.
- Cloudflare signing is local crypto — no external call, no timeout concern; the CF PEM never logs (`TestNoSecretsInLogs` gained `-----BEGIN.*PRIVATE KEY` in spec B3 — lands here with the PEM's first user).
- DDL through `schema.Engine`/migration `0012`; errors via `WriteError`; vocabulary +`commerce.entitlement.grant`, `video.token.issue`.
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V3-2 — run before Task 1)

- [ ] Confirm: the field-type registry and `schema.CanConvert(from Field, to FieldType, …)` matrix shape (V1-P3 T3) — `video` joins as a converts-to/from-nothing row; `createTableDDL`'s type→column mapping (`video`→`TEXT`).
- [ ] Confirm `content.Set`'s per-type validation table (V1-P4 T3) — `video` values are provider-UID strings (`^[A-Za-z0-9]{16,64}$` for CF Stream; re-validate against real UIDs).
- [ ] Confirm `commerce.OrderMachine.SetPaidHook` and `store.CmsOrder.LineItems` JSONB shape (V3-P3 T4/T1).
- [ ] Confirm `app.Config.Video.{KeyID, KeyPEM, CustomerCode}` (V3-P1 T1) and `RequireVideo`; the V1-P5 RS256 signing primitives (reuse the parse/sign helpers rather than re-implementing).
- [ ] Confirm the public read gate's anonymous-404 / authenticated-403 split (V1-P6) — the token endpoint layers the 403 entitlement gate AFTER the record-visibility gate.
- [ ] Migrations through `0011`; `0012` next.

## File Structure

```
internal/schema/…                        video field type (types/ddl/convert/validate — modify)
internal/content/document.go             video UID validation (modify)
internal/store/migrations/0012_entitlements.sql
internal/store/queries/entitlements.sql  grant, check, backfill (sqlc)
internal/video/provider.go               Provider interface + PlaybackGrant
internal/video/cfstream.go               Cloudflare Stream adapter
internal/commerce/entitlements.go        paid-hook + backfill
internal/httpapi/video_handlers.go       token endpoint
web/src/lib/components/widgets/Video.svelte
web/e2e/v3p4-video.spec.ts
docs/BUSINESS_RULES.md                   + BR-VID-1; BR-AUTH-15 entitlements assertion
docs/architecture/03-dynamic-schema.md   field-type reference row (amend)
docs/architecture/04-api-layer.md        token route (amend)
docs/architecture/05-auth-security.md    playback-token threat note (amend)
docs/architecture/07-data-model.md       video mapping + cms_entitlements row w/ D-V3-10 rationale (amend)
```

---

### Task 1: `video` field type end-to-end

**Files:**
- Modify: `internal/schema/types.go` (+`FieldConfig.Paywalled bool`, JSON `paywalled`), `internal/schema/ddl.go` (`video`→`TEXT`), `internal/schema/convert.go` (closed row), `internal/schema/validate.go` (paywalled legal only on video), `internal/content/document.go` (UID shape validation), record serialization (both scopes: video value → `{"uid": <string|null>, "paywalled": bool}`), `docs/architecture/03-dynamic-schema.md`, `docs/architecture/07-data-model.md` (mapping row)
- Test: `internal/schema/video_field_test.go`, extend content/read suites

**Interfaces:**
- Produces: field type `video`; `Document.Set` accepts a UID string matching `^[A-Za-z0-9]{16,64}$` or null (else 422 naming the field); every read (admin, public, list, single, search projections) serializes `{"uid", "paywalled"}`; `paywalled` on a non-video field → engine validation error.

- [ ] **Step 1: Failing tests** — create a collection with `trailer(video)` + `full(video, paywalled)`; anatomy: both columns `TEXT`; Set: valid UID round-trips, `"not a uid!"` → 422, null clears; reads: public + admin responses carry the object shape with correct `paywalled` flags and **no URL-shaped string anywhere** (`TestBR_VID_1_SerializationNeverContainsPlaybackURL`: regex `cloudflarestream\.com` over the full response body → no match); convert video→text → `ErrBadConversion`; `paywalled` on a `number` field → 422.
- [ ] **Step 2:** FAIL. **Step 3:** implement. **Step 4:** PASS.
- [ ] **Step 5: Amend docs** — 03 field-type table: `video → TEXT (provider UID; config paywalled — V3)` present tense; 07 user-field mapping row likewise.
- [ ] **Step 6: Commit** — `git commit -m "feat: video field type with paywalled flag (F-30, BR-VID-1 read half)"`

---

### Task 2: `internal/video` — Provider + Cloudflare Stream adapter

**Files:**
- Create: `internal/video/provider.go`, `internal/video/cfstream.go`
- Modify: V1 secret-log test (+PEM regex)
- Test: `internal/video/cfstream_test.go`

**Interfaces:**
- Produces:
  - `video.Provider interface { PlaybackToken(ctx context.Context, uid string, ttl time.Duration) (PlaybackGrant, error) }`; `video.PlaybackGrant{Token string; URL string; ExpiresAt time.Time}`.
  - `video.NewCFStream(keyID string, keyPEM []byte, customerCode string) (*CFStream, error)` — parses the RSA private key at construction (fail fast). `PlaybackToken`: JWT header `{"alg":"RS256","kid":<keyID>}`, claims `{"sub":<uid>,"exp":<unix now+ttl>}`, RS256-signed; `URL = "https://customer-<code>.cloudflarestream.com/<token>/manifest/video.m3u8"`; `ExpiresAt` from the claim. TTL is caller-supplied; the adapter rejects ttl > 15 min (`ErrTTLTooLong` — BR-VID-1's cap enforced at the lowest layer).

- [ ] **Step 1: Failing tests** — decode the produced JWT with the matching public key: header kid exact, `sub` = uid, `exp` within ±2 s of now+10 m; URL golden (`customer-abc123.cloudflarestream.com/<token>/manifest/video.m3u8`); ttl 20 m → `ErrTTLTooLong`; bad PEM at construction → error; secret-log suite green with `-----BEGIN.*PRIVATE KEY` regex added.
- [ ] **Step 2:** FAIL. **Step 3:** implement (reuse V1-P5's RS256 sign helper if exported; else 30 lines of `crypto/rsa` + `encoding/base64` JWT assembly). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: video provider interface + Cloudflare Stream signed tokens (D-V3-4)"`

---

### Task 3: Entitlements — migration 0012, paid-hook, backfill

**Files:**
- Create: `internal/store/migrations/0012_entitlements.sql` (spec §D4 DDL verbatim), `internal/store/queries/entitlements.sql`, `internal/commerce/entitlements.go`
- Modify: `internal/app/app.go` (hook registration + boot backfill), V2-P8 erasure test (+entitlements assertion), `internal/audit/actions.go` (+`commerce.entitlement.grant`), `docs/architecture/07-data-model.md`
- Test: `internal/commerce/entitlements_test.go`

**Interfaces:**
- Produces:
  - sqlc: `GrantEntitlement(end_user_id, record_id, order_id)` (`ON CONFLICT (end_user_id, record_id) DO NOTHING`), `HasEntitlement(end_user_id, record_id) (bool)`, `BackfillEntitlements()` (one INSERT…SELECT over `cms_orders WHERE status='paid'`, unnesting `line_items->record_id`, `ON CONFLICT DO NOTHING`).
  - `commerce.EntitlementHook(q *store.Queries, rec *audit.Recorder) OrderPaidHook` — inside the paid transaction: one grant per distinct `line_items[].record_id`, audit `commerce.entitlement.grant` per new row (detail: record_id, order_id).
  - `app.Run` registers the hook on the `OrderMachine` and runs `BackfillEntitlements` once per boot after migrations (idempotent; full-scan cost acceptable at design load — bounded-scan is a future optimization, not a contract).

- [ ] **Step 1: Failing tests** — paid webhook (V3-P3 harness) → entitlement rows for both line items, audit events present; replayed webhook → no duplicate rows (conflict) and no duplicate grant audits (hook not re-run — V3-P3's idempotency); backfill over a pre-seeded paid order without rows → rows appear; second backfill run → unchanged; erasure test extended: entitlements vanish by CASCADE.
- [ ] **Step 2:** FAIL. **Step 3:** implement.
- [ ] **Step 4:** PASS. **Step 5: Amend 07** — `cms_entitlements` matrix row carrying the D-V3-10 rationale verbatim ("the only durable representation of who may play what; deriving from line_items JSONB per request is neither indexable nor auditable"); note it as an addition beyond the originally reserved names.
- [ ] **Step 6: Commit** — `git commit -m "feat: purchase entitlements — paid-hook grants + idempotent backfill (D-V3-6)"`

---

### Task 4: Token endpoint (BR-VID-1)

**Files:**
- Create: `internal/httpapi/video_handlers.go`
- Modify: public router (under `RequireVideo`), `internal/audit/actions.go` (+`video.token.issue`), `docs/BUSINESS_RULES.md` (+BR-VID-1), `docs/architecture/04-api-layer.md`, `docs/architecture/05-auth-security.md`
- Test: `internal/httpapi/video_handlers_test.go`

**Interfaces:**
- Produces: `GET /api/v1/collections/{slug}/records/{id}/video/{field}/token` — gate order:
  1. `RequireVideo` (video-less → 503).
  2. Record visibility exactly as a public single read (`Decide(read)`, publish floor, anonymous-deny → 404 / authenticated-deny → 403 — the V1 split).
  3. `{field}` must be a `video` field with a non-null UID (else 404 `not_found` naming the field).
  4. `paywalled=false` → grant for any principal that passed step 2.
  5. `paywalled=true` → principal must be an end user (anonymous → **403 `forbidden`** with `details:{"reason":"entitlement_required"}` — UAC-3.2's explicit code; the record is visible, playback is gated) AND `HasEntitlement(user, record)` (missing → same 403).
  6. → `Provider.PlaybackToken(uid, 10*time.Minute)` → `{data:{token, url, expires_at}}`, `Cache-Control: no-store` always; audit `video.token.issue` (record, field, principal — **never the token**).

- [ ] **Step 1: Failing tests** — **`TestUAC_3_2_PaywalledTokenArc`** (integration): publish a record with a paywalled video; anonymous token request → 403 `entitlement_required`; a registered end user without purchase → 403; run the V3-P3 paid-webhook fixture for that user/record → token request → 200, JWT decodes with `sub`=uid and `exp`≈now+10 m (expiry half of UAC-3.2 asserted by decoding — the "same token fails after expiry" clause is the provider's enforcement, proven at the manual gate); non-paywalled field → anonymous gets a grant; unpublished record → anonymous 404; `no-store` present; token absent from logs and audit detail.
- [ ] **Step 2:** FAIL. **Step 3:** implement + BUSINESS_RULES BR-VID-1 (spec §D4 text) + 04 route block + 05 threat note (token leakage bounded by TTL; tokens are bearer — `no-store` + short TTL are the mitigations). **Step 4:** PASS (`make trace`: BR-VID-1 traces). **Step 5: Commit** — `git commit -m "feat: entitlement-gated signed playback tokens (BR-VID-1, UAC-3.2)"`

---

### Task 5: UI — video widget + field settings, e2e + manual gate step

**Files:**
- Create: `web/src/lib/components/widgets/Video.svelte`, `web/e2e/v3p4-video.spec.ts`
- Modify: `web/src/routes/RecordEdit.svelte` (widget dispatch), `web/src/routes/CollectionEdit.svelte` (paywalled checkbox on video fields)

- [ ] **Step 1: Failing e2e** — schema half: add a `video` field with Paywalled checked; content half: paste a UID into the widget, save, publish; public read (`page.request`) shows `{uid, paywalled:true}` and no `cloudflarestream.com` string; anonymous token request via `page.request` → 403; (entitled-arc stays at the Go level — e2e has no Stripe fixture).
- [ ] **Step 2: Implement** — `Video.svelte`: UID text input with the shape hint, paywalled badge when the field config says so; CollectionEdit: Paywalled checkbox on video-type fields.
- [ ] **Step 3:** e2e PASS.
- [ ] **Step 4: Record the manual gate step** — for the V3-P5 gate checklist: "UAC-3.2 manual verification: with real CF_STREAM_* config and a real uploaded video UID, request a token as an entitled test user, play the returned URL in a browser (HLS manifest loads), then retry the same URL after 10 minutes and confirm playback fails."
- [ ] **Step 5: Commit** — `git commit -m "feat: video field widget and paywalled setting (UAC-3.2)"`

---

### Task 6: Acceptance sweep

- [ ] `make test && make trace` green; waiver empty; BR-VID-1 traces.
- [ ] Full e2e suite green (V1 + V2 + v3p1–p4); `make bench` unchanged.
- [ ] `git grep -rn "cloudflarestream" internal/ | grep -v "video/\|_test"` → empty (URL construction confined to the adapter).
- [ ] Video-less boot: token route → 503 `{"reason":"video_disabled"}`; everything else unaffected.
- [ ] Doc amendments landed: BUSINESS_RULES (BR-VID-1 + BR-AUTH-15 assertion), 03, 04, 05, 07 (D-V3-10 rationale).

## Self-Review Notes (execution-time attention)

- **The 403-on-paywalled decision** (vs the V1 anonymous-404 posture) is UAC-3.2's literal requirement and is safe: the record's *existence* is already public (it's published); only playback is gated. The gate-order list in Task 4 is normative — visibility 404s still fire first for unpublished records.
- **Token expiry is provider-enforced**; the binary's contribution is the `exp` claim and the ≤ 15 min cap. The automated suite proves the claim; only the manual step proves CF honors it — that split is recorded honestly in the gate.
- **Backfill runs every boot** — idempotent and full-scan; if order volume ever makes this measurable, bound it by `created_at > max(entitlements.created_at) - interval '7 days'` — an optimization explicitly deferred.
- **Anonymous grants on non-paywalled fields** still require record visibility — a draft's trailer is NOT publicly tokenizable; the Task 4 test covers it.
