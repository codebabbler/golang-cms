# V3 Implementation Phasing — Design

**Date:** 2026-07-14 · **Status:** Approved pending user review · **Scope:** Version 3 (Commerce & Video) of golang-cms per `docs/architecture/11-roadmap.md`

## Context

V1 and V2 are fully planned (seventeen plans in `docs/superpowers/plans/v1/` and `v2/`) but not yet executed. V3 planning happens now, producing execution plans only. Three facts shape this spec:

1. **The docs are thinnest for V3.** Only `cms_carts` and `cms_orders` are reserved in `07-data-model.md`; **zero V3-tagged BRs exist** — every V3 invariant needs a new BR-ID shipped in its phase (the D-V2-8 pattern). This spec is the delivery-cycle design record for every V3 layout and contract.
2. **External SaaS is sanctioned; new configuration is unavoidable.** O-9 excludes Redis/queues/caches, not HTTPS integrations — Stripe and the video provider are API integrations, not runtime dependencies (BR-RUNTIME-2 holds). V3 necessarily adds env vars; they follow the `S3_*` precedent: atomic groups with graceful absence.
3. **O-8 fixes the video posture:** transcoding and packaging are fully delegated; **the binary signs playback tokens only.**

V3 plans are written against V1+V2 plans, not code — the D-V1-3 re-validation rule binds every plan's execution.

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-V3-1 | Five focused phases in dependency order: foundation → carts → Stripe/orders → video paywalls → Tiptap blocks + gate | Dependencies are linear by construction (carts need commerce config; orders consume carts; entitlements derive from paid orders; video blocks need the video phase). Carts and Stripe stay separate so the first two phases run with no external account and the money-touching diff reviews alone. |
| D-V3-2 | Inherit the V2 planning pattern wholesale: designs at implementation grade in this spec + per-plan doc amendments; vertical slices (backend + admin UI + e2e per phase); all plans authored up front in `docs/superpowers/plans/v3/` with re-validation preambles; docs win over drifted plan detail | Proven groove; V3's shape (few features, thin docs) fits it. |
| D-V3-3 | Stripe via a thin hand-rolled REST client (`internal/stripe`) — no SDK | Two API surfaces (checkout session create, webhook signature verification) don't justify a large dependency; smaller audited surface; house dependency posture. |
| D-V3-4 | Video via `internal/video` `Provider` interface + one reference adapter: Cloudflare Stream signed tokens | Ecosystem-consistent (R2 and Cloudflare Image Resizing are already the blessed delegates — O-7); one real, testable adapter; the interface keeps other vendors possible. |
| D-V3-5 | Products are designated collections: `commerce_config` JSONB on `cms_collections` names the price field and pins the currency | The catalog stays config-driven — schema engine, search, media, revisions all work on products for free; no parallel CRUD. |
| D-V3-6 | Entitlement unit = the purchased record itself: buying product record P entitles the buyer to P's paywalled video fields | No indirection; one `UNIQUE(end_user_id, record_id)` row per purchase; UAC-3.2 maps directly. Grants-relation bundles are deferred (YAGNI). |
| D-V3-7 | New env vars arrive as atomic groups with graceful absence: `STRIPE_SECRET_KEY`+`STRIPE_WEBHOOK_SECRET` (pair; absent → commerce-less: cart/checkout/order/webhook routes 503 `unavailable`) and `CF_STREAM_KEY_ID`+`CF_STREAM_KEY_PEM`+`CF_STREAM_CUSTOMER_CODE` (trio; absent → video-less: token routes 503) | The exact `S3_*`/media-less precedent (BR-MEDIA-6 pattern); partial groups fail startup listing the missing names. Supersedes V2's "no new env vars" rule for V3 scope only. |
| D-V3-8 | New BR-IDs ship with their phases: BR-COM-1/2 (V3-P3), BR-VID-1 (V3-P4); the V3 harness flip at V3-P1 drops trace.sh's now-dormant V3 exemption; the waiver file stays empty throughout V3 | Rule + enforcement point + tests land together; nothing to waive. |
| D-V3-9 | Stripe webhook idempotency via state-machine transitions, not an event-dedup table | Replayed events map to no-op transitions (`paid → paid`); keeps the table count at 07's reserved two plus entitlements; simpler recovery. |
| D-V3-10 | `cms_entitlements` is added beyond 07's reserved-names list | Entitlements are the only durable representation of "who may play what"; deriving them from `line_items` JSONB on every token request is neither indexable nor auditable. V3-P4 ships the 07 amendment with this rationale. |

## A. Phase Map (binding sequence)

All phases assume V1+V2 complete (gates green). Migrations continue the ledger (V2 ends at 0008).

| Phase | Scope | Migration | UAC / BR anchors | Exit gate |
|---|---|---|---|---|
| **V3-P1 Commerce foundation** | V3 env groups parsed in full with 503-absence modes (D-V3-7); `commerce_config` on `cms_collections` + validation + admin UI section; **V3 harness flip** | 0009 | F-29/F-30 groundwork | 503 modes proven; config round-trips; trace green, waiver empty |
| **V3-P2 Carts** | `cms_carts` (FK CASCADE erasure), JWT-scoped cart API, no stored prices | 0010 | F-28 | Cart arc e2e; BR-AUTH-15 test extended to carts |
| **V3-P3 Stripe checkout + orders** | `internal/stripe` client; `cms_orders` with checkout-time price snapshot; checkout + webhook endpoints; `OrderPaidHook` seam; orders admin screen; **BR-COM-1/2** | 0011 | **UAC-3.1**; F-29 | Stub-verified checkout + signed webhook replay; manual Stripe test-mode gate step documented |
| **V3-P4 Video paywalls** | `video` field type; `Provider` + Cloudflare Stream adapter; `cms_entitlements` + paid-hook writes + backfill; token endpoint (anonymous paywalled → **403**); **BR-VID-1** | 0012 | **UAC-3.2**; F-30 | Token arc e2e (403 / signed / expiry); manual real-playback gate step documented |
| **V3-P5 Tiptap blocks + V3 gate** | `productBlock`/`videoBlock` nodes end-to-end (validation, editor node views, public resolution, HTML rendering); **V3 gate**: UAC-3.1…3.3 + V1/V2 gates green | — | **UAC-3.3**; F-31 | Full programme sweep |

**Dependencies:** strictly linear P1→P2→P3→P4→P5 (P5 needs P1 for product blocks and P4 for video blocks).

## B. Cross-Phase Rules

Rules B1–B7 of the V2 spec apply verbatim (re-validation preamble, audit continuity, migrations ledger, secrets discipline, `main`, gates), with these V3 additions:

1. **Money is integer minor units.** Prices and amounts are `BIGINT` cents server-side, everywhere. The commerce price field must be a `number` with scale 0; currency is pinned per collection (ISO 4217 lowercase, the Stripe convention); carts are single-currency — a mixed-currency add → 422 naming the conflict.
2. **External-call posture.** Stripe calls carry a 10 s timeout (the S3 precedent); Cloudflare token signing is local crypto (no call). Neither integration is retried by V3 code beyond what the user retriggers — Stripe retries its own webhooks.
3. **Secrets discipline extends.** `TestNoSecretsInLogs` gains `sk_(test|live)_[A-Za-z0-9]{10,}`, `Stripe-Signature`, and `-----BEGIN.*PRIVATE KEY` (the CF PEM); the Stripe webhook secret matches the existing `whsec_` regex from V2-P6.
4. **New vocabulary domains:** `commerce.cart.update|clear`, `commerce.order.create|paid|failed|expired`, `commerce.entitlement.grant`, `video.token.issue` (issue events audit with record/field, never the token), `schema.commerce_config.update`.
5. **Erasure completeness accrues.** BR-AUTH-15's test extends each phase: carts vanish by FK CASCADE (V3-P2), orders survive with `end_user_id` rewritten to the tombstone — financial history retained sans identity (V3-P3, BR text amendment), entitlements vanish by FK CASCADE (V3-P4).

## C. Programme Acceptance Criteria

1. Five plan files in `docs/superpowers/plans/v3/`, each with the re-validation preamble and an acceptance sweep.
2. UAC mapping: 3.1→P3, 3.2→P4, 3.3→P5; the two documented **manual gate steps** (real Stripe test-mode checkout; real Cloudflare Stream playback) appear in P3/P4 plans and are re-run at the P5 gate.
3. Waiver file empty at every phase exit (no V3 rule is ever waived).
4. Every §D layout owned by exactly one phase plan + doc amendment (07 gains `cms_carts`, `cms_orders`, `cms_entitlements` rows; 04 gains the new public routes; 05 gains the Stripe webhook + playback-token threat notes; 08 gains the new vocabulary; 09 gains the env groups).
5. No V1/V2 contract reopened; named extension points only: `content.Set` richText allowlist (P5), `richtext.RenderHTML` allowlist (P5), `BR-AUTH-15` text (P3), `TestNoSecretsInLogs` regex list (P3), config loader groups (P1), field-type registry (P4 `video`).
6. V1 and V2 gates stay green at every phase exit.

## D. Design Record (implementation grade)

### D1. Commerce foundation (V3-P1)

- **Env (D-V3-7).** `STRIPE_SECRET_KEY` (`sk_test_…`/`sk_live_…`), `STRIPE_WEBHOOK_SECRET` (`whsec_…`); `CF_STREAM_KEY_ID`, `CF_STREAM_KEY_PEM` (base64-encoded PEM, decoded at load), `CF_STREAM_CUSTOMER_CODE`. Partial group at startup → the V1 `ConfigError` listing missing names. `app.Config` gains `Commerce *CommerceConfig` / `Video *VideoConfig` (nil = mode off). Route guards return 503 `unavailable` with `details: {"reason": "commerce_disabled"}` / `"video_disabled"` (the media-less shape).
- **`commerce_config`** (migration 0009: `ALTER TABLE cms_collections ADD COLUMN commerce_config JSONB`): `{"enabled": true, "priceField": "price", "currency": "usd"}`. Validation (schema engine, beside `access_rules`): `priceField` names an existing `number` field with scale 0; `currency` ∈ ISO 4217 (lowercase, closed list in code); disabling with live cart references is legal (carts re-validate at read/checkout). Field drop/type-change of the named price field → the engine rejects with the existing inbound-reference error pattern naming `commerce_config`.
- **Harness flip.** `scripts/trace.sh` drops the `\(V3\)` exemption clause entirely (it matches zero rules — hygiene). Waiver untouched (empty).
- **Admin UI.** CollectionEdit gains a "Commerce" section: enable toggle, price-field select (filtered to eligible fields), currency select.

### D2. `cms_carts` (V3-P2)

```sql
CREATE TABLE cms_carts (
    id          UUID PRIMARY KEY,                    -- UUIDv7
    end_user_id UUID NOT NULL UNIQUE REFERENCES cms_end_users(id) ON DELETE CASCADE,
    currency    TEXT NOT NULL,
    items       JSONB NOT NULL DEFAULT '[]',         -- [{record_id, collection_id, qty}]
    created_at  TIMESTAMPTZ NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL
);
```

- **API (end-user JWT required; anonymous → 401):** `GET /api/v1/cart` (hydrates each item at read time: `display` = the record's first `text` field value by position (empty string if the collection has none), current price, published-status; a since-unpublished or de-commerced item is flagged `"stale": true` in the response, never silently dropped); `POST /api/v1/cart/items {collection, record_id, qty}`; `PATCH /api/v1/cart/items/{record_id} {qty}`; `DELETE /api/v1/cart/items/{record_id}`; `DELETE /api/v1/cart`.
- **Validation:** record exists, is published, collection commerce-enabled; `qty` 1–99; ≤ 50 distinct items; first add pins the cart's `currency` from the collection's config — later adds with a different currency → 422 naming both currencies.
- **No stored prices** — the cart stores references; money is read from records at display and snapshotted only at checkout (BR-COM-1's write side lives in P3).
- **Erasure:** FK CASCADE deletes the cart with the user; the BR-AUTH-15 test gains the assertion.

### D3. Stripe checkout + orders (V3-P3)

```sql
CREATE TABLE cms_orders (
    id                UUID PRIMARY KEY,              -- UUIDv7
    end_user_id       UUID NOT NULL,                 -- no FK: erasure rewrites to tombstone
    stripe_session_id TEXT UNIQUE,
    status            TEXT NOT NULL CHECK (status IN ('pending','paid','failed','expired')),
    currency          TEXT NOT NULL,
    amount_total      BIGINT NOT NULL,
    line_items        JSONB NOT NULL,   -- [{record_id, collection_id, title, unit_amount, qty}]
    created_at        TIMESTAMPTZ NOT NULL,
    updated_at        TIMESTAMPTZ NOT NULL
);
CREATE INDEX ix_cms_orders_user ON cms_orders (end_user_id, created_at DESC);
```

- **`internal/stripe` client (D-V3-3):** `New(secretKey string, opts ...Option) *Client` (`WithBaseURL` is the test seam — never an env var); `(*Client).CreateCheckoutSession(ctx, p SessionParams) (SessionResult, error)` — form-encoded POST `/v1/checkout/sessions`, `mode=payment`, inline `price_data` per line item (currency, `unit_amount`, `product_data[name]`), `metadata[order_id]`, `success_url`/`cancel_url` from the request body (validated absolute http(s) — the consuming site's pages); returns `{ID, URL}`. `stripe.VerifySignature(payload []byte, header, secret string, now time.Time) error` — parses `t=…,v1=…`, HMAC-SHA256 over `t + "." + payload`, constant-time compare, tolerance 5 min (**BR-COM-2** half).
- **Checkout:** `POST /api/v1/checkout {success_url, cancel_url}` (end-user JWT; commerce mode on): load cart (empty → 422); re-validate every item (stale item → 422 naming it); **snapshot prices server-side from the records into `line_items` and `amount_total`** (**BR-COM-1**: client-supplied amounts do not exist in this flow, and the snapshot is what Stripe is told to charge); insert `pending` order + create the Stripe session (session create failure → order deleted, 502-mapped `unavailable`); respond `{data:{order_id, url}}`.
- **Webhook:** `POST /api/v1/stripe/webhook` — raw-body read (before any JSON decoding), `VerifySignature` mandatory (fail → 400, logged without the signature); handled events: `checkout.session.completed` → order (by `metadata.order_id`) `pending→paid`, clear the user's cart, run `OrderPaidHook`; `checkout.session.expired` → `pending→expired`; `checkout.session.async_payment_failed` → `pending→failed`. **Idempotency (D-V3-9):** transitions from a non-`pending` state are audited no-ops returning 200. Unknown event types → 200 (acknowledged, ignored).
- **`OrderPaidHook func(ctx, tx pgx.Tx, order Order) error`** — registered by V3-P4; runs inside the paid transition's transaction (the V2-P6 TxHook pattern; hook error rolls back the transition, Stripe redelivers).
- **Erasure:** `auth.EraseEndUser` gains `UPDATE cms_orders SET end_user_id=$tomb WHERE end_user_id=$1`; BR-AUTH-15 text amended ("orders are retained with tombstoned identity").
- **Admin UI:** Orders screen — list (status filter, keyset cursor on `(created_at,id)`), detail drawer with line items; read-only (order state belongs to Stripe's webhook stream).
- **New BRs:** **BR-COM-1** "Order amounts are snapshotted server-side from collection records at checkout; no client-supplied amount is ever read. Enforced in the checkout handler and `internal/stripe.SessionParams` (which has no client-amount field)." **BR-COM-2** "Every Stripe webhook is signature-verified (HMAC over `t.payload`, 5-minute tolerance) before parsing, and order transitions are idempotent. Enforced in `stripe.VerifySignature` + the order state machine."

### D4. Video paywalls (V3-P4)

```sql
CREATE TABLE cms_entitlements (
    id          UUID PRIMARY KEY,                    -- UUIDv7
    end_user_id UUID NOT NULL REFERENCES cms_end_users(id) ON DELETE CASCADE,
    record_id   UUID NOT NULL,
    order_id    UUID NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    UNIQUE (end_user_id, record_id)
);
```

- **`video` field type:** stored `TEXT` (the provider video UID); `FieldConfig` gains nothing new beyond `Paywalled bool` (JSON `paywalled`, legal only on `video` fields). Conversion matrix: `video` converts to/from nothing (new closed row). Serialization: video fields render as `{"uid": …, "paywalled": bool}` — **never a playback URL** (BR-VID-1's read-path half).
- **Provider (D-V3-4):** `video.Provider interface { PlaybackToken(ctx, uid string, ttl time.Duration) (PlaybackGrant, error) }`; `PlaybackGrant{Token string; URL string; ExpiresAt time.Time}`. Cloudflare Stream adapter: RS256 JWT — header `{alg, kid: CF_STREAM_KEY_ID}`, claims `{sub: uid, exp}` — signed with the decoded `CF_STREAM_KEY_PEM`; `URL = https://customer-<code>.cloudflarestream.com/<token>/manifest/video.m3u8`. TTL fixed 10 min (**BR-VID-1** caps ≤ 15 min).
- **Entitlements:** the `OrderPaidHook` inserts one row per distinct `line_items[].record_id` (`ON CONFLICT DO NOTHING`); a startup backfill task (one-shot, idempotent, inside V3-P4's first boot — not a migration, it's data) grants for orders already `paid`. Erasure: FK CASCADE + BR-AUTH-15 assertion.
- **Token endpoint:** `GET /api/v1/collections/{slug}/records/{id}/video/{field}/token` (video mode on, else 503): field must exist and be `video` type (else 404-shape parity with record reads); record must be readable under the requester's `Decision` (the standard public gate first — an invisible record stays 404/403 per the V1 split); then: `paywalled=false` → grant for anyone who can read the record; `paywalled=true` → end-user JWT required and entitlement row must exist, else **403 `forbidden`** (UAC-3.2's explicit code — anonymous paywalled requests get 403, not 404, because the record itself is visible; only playback is gated); grant → `{data:{token, url, expires_at}}`, `Cache-Control: no-store` always (grants are principal-specific). Audit `video.token.issue` (record/field/user — never the token).
- **New BR:** **BR-VID-1** "Paywalled video playback is granted exclusively through short-lived (≤ 15 min) signed tokens issued to entitled end users; the binary signs tokens and never proxies or stores video bytes (O-8). Video field serialization never contains a playback URL. Enforced in the token endpoint + `video.Provider` adapters."
- **07 amendment (D-V3-10):** `cms_entitlements` row added with the decision rationale.

### D5. Tiptap blocks (V3-P5)

- **Node schemas (canonical JSONB):** `{"type":"productBlock","attrs":{"collectionId":"<uuid>","recordId":"<uuid>"}}`; `{"type":"videoBlock","attrs":{"collectionId":"<uuid>","recordId":"<uuid>","field":"<slug>"}}`. Both are leaf nodes (no `content`).
- **Write validation:** `content.Set`'s richText walk allowlist gains both types with attr shape validation (UUID strings; `field` a legal slug). Referential existence is **not** checked at write (blocks must survive later deletions — UAC-3.3; same philosophy as V1 relation shape-checks).
- **Public resolution (read time):** the response serialization walk (V2-P4's seam) collects block attrs across the page, resolves with **one batched query per block type** through the builder's anonymous-strict published-only form (the ExpandRelations posture): `productBlock` → `"resolved": {"id", "collectionSlug", "data": {visible projected fields incl. price field}, "seo": …}`; `videoBlock` → `"resolved": {"uid", "paywalled"}` (never a token — the client calls the token endpoint). Deleted/trashed/unpublished/invisible target → `"resolved": null`, the node otherwise intact — **the post never breaks** (UAC-3.3). Admin reads (`format=json`) get raw canon (the editor resolves via its own admin fetches).
- **HTML rendering:** `richtext.RenderHTML` allowlist gains both as attribute-only placeholders — `<figure data-cms-block="product" data-collection-id="…" data-record-id="…"></figure>` / `<figure data-cms-block="video" data-record-id="…" data-field="…"></figure>` (attrs escaped; no body text — the renderer stays a pure function with no data access; the consuming site hydrates from the attributes).
- **Editor:** Tiptap node extensions + Svelte node views in `RichText.svelte`: toolbar "Insert product" (record picker over commerce-enabled collections) and "Insert video" (record + video-field picker); rendered as cards in the editor; delete = node delete.
- **V3 gate:** UAC-3.1…3.3 automated arcs + the two manual gate steps re-run (D-V3 gate task lists them verbatim).

## E. Out of Scope (V3)

Everything in REQUIREMENTS §5 (O-1…O-12) — notably in-binary transcoding (O-8) and plugin runtimes (O-11); subscriptions/recurring billing; refunds and partial captures (order states model the checkout session lifecycle only — refunds are Stripe-dashboard operations in V3); multi-currency carts; grants-relation entitlement bundles (D-V3-6); tax/VAT calculation (Stripe Checkout's tax features are not enabled); webhook endpoints for providers other than Stripe; video analytics.
