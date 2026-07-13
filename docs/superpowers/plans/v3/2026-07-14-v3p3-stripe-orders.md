# V3-P3 — Stripe Checkout + Orders Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Cart → Stripe test-mode checkout → persisted order with matching line items (UAC-3.1, F-29), through a thin hand-rolled client, with server-side price snapshots (new BR-COM-1) and signature-verified idempotent webhooks (new BR-COM-2). **This is the money-touching phase — review accordingly.**

**Architecture:** `internal/stripe` is ~200 lines: one form-encoded POST (checkout sessions) and one HMAC verifier (`Stripe-Signature`), with an injectable base URL as the only test seam. Checkout snapshots prices from records into `cms_orders.line_items` inside the pending-order insert; the webhook drives the order state machine (`pending → paid|failed|expired`) whose transitions are idempotent by construction (D-V3-9 — no event table). The paid transition clears the cart and runs the `OrderPaidHook` seam V3-P4 registers.

**Tech Stack:** Go 1.25, pgx/v5, sqlc, HMAC-SHA256, Svelte 5, Playwright.

## Global Constraints (spec: `docs/superpowers/specs/2026-07-14-v3-implementation-phasing-design.md`)

- Authority chain: `docs/BUSINESS_RULES.md` > `docs/architecture/*` > skills > code.
- **BR-COM-1 (added this phase):** order amounts are snapshotted server-side from collection records at checkout; no client-supplied amount is ever read — `stripe.SessionParams` has no client-amount field.
- **BR-COM-2 (added this phase):** every Stripe webhook is signature-verified (HMAC over `t.payload`, 5-minute tolerance, constant-time compare) before parsing; order transitions are idempotent.
- Money integer minor units; Stripe calls 10 s timeout; secrets discipline: `TestNoSecretsInLogs` += `sk_(test|live)_[A-Za-z0-9]{10,}`, `Stripe-Signature` (spec B3).
- Migration `0011`; vocabulary +`commerce.order.create|paid|failed|expired`. Errors via `WriteError`; no SDK dependency.
- Done = `make test && make trace` green plus this plan's acceptance sweep.

## Re-Validation Preamble (D-V1-3 / D-V3-2 — run before Task 1)

- [ ] Confirm: `commerce.CartService.Get` + staleness semantics and `commerce.HydratedItem{…UnitAmount int64…}` (V3-P2 T2); `RequireCommerce` + end-user gate (V3-P1/P2); `app.Config.Commerce.{SecretKey,WebhookSecret}` (V3-P1 T1).
- [ ] Confirm the V2-P6 `TxHook` post-commit-nudge resolution — `OrderPaidHook` reuses whatever seam shape execution chose there.
- [ ] Confirm `auth.EraseEndUser`'s transaction body (V2-P8 T4) — Task 4 adds the orders rewrite inside it.
- [ ] Confirm the raw-body handling pattern for signature-verified endpoints (none exists yet — the webhook handler must read the body BEFORE any middleware/decoder consumes it; check the V1 body-cap middleware ordering).
- [ ] Migrations through `0010`; `0011` next.

## File Structure

```
internal/store/migrations/0011_orders.sql
internal/store/queries/orders.sql          create, get, transition, list, tombstone (sqlc)
internal/stripe/client.go                  CreateCheckoutSession (form-encoded POST)
internal/stripe/signature.go               VerifySignature
internal/commerce/checkout.go              snapshot + order + session orchestration
internal/commerce/orders.go                state machine + OrderPaidHook
internal/httpapi/checkout_handlers.go      POST /api/v1/checkout, POST /api/v1/stripe/webhook
web/src/routes/Orders.svelte               admin read-only screen
web/e2e/v3p3-orders.spec.ts
docs/BUSINESS_RULES.md                     + Commerce section: BR-COM-1/2; BR-AUTH-15 amendment
docs/architecture/04-api-layer.md          checkout/webhook routes (amend)
docs/architecture/05-auth-security.md      webhook threat note (amend)
docs/architecture/07-data-model.md         cms_orders row (amend)
```

---

### Task 1: Migration 0011 + sqlc + 07 amendment

**Files:**
- Create: `internal/store/migrations/0011_orders.sql` (spec §D3 DDL verbatim, incl. `ix_cms_orders_user`), `internal/store/queries/orders.sql`
- Modify: `docs/architecture/07-data-model.md`
- Test: `internal/store/orders_test.go`

**Interfaces:**
- Produces sqlc: `CreateOrder`, `GetOrder(id)`, `GetOrderBySession(stripe_session_id)`, `SetOrderSession(id, session_id)`, `TransitionOrder(id, from, to) (rowsAffected)` (`UPDATE cms_orders SET status=$3, updated_at=now() WHERE id=$1 AND status=$2` — the zero-rows result IS the idempotency signal), `ListOrders(status, keyset (created_at,id) desc)`, `DeleteOrder(id)` (checkout-failure cleanup of `pending` only — `AND status='pending'`), `TombstoneOrdersForUser(end_user_id, tombstone)`.

- [ ] **Step 1: Failing test** — migrate; create → get; `TransitionOrder(pending→paid)` affects 1 row, repeat affects 0 (idempotency primitive); status CHECK rejects `refunded`; session unique.
- [ ] **Step 2:** FAIL. **Step 3:** implement; `make generate`. **Step 4:** PASS.
- [ ] **Step 5: Amend 07** — `cms_orders` matrix row ("line items snapshot prices at checkout — BR-COM-1; `end_user_id` has no FK: erasure rewrites it to the tombstone, financial history retained"); drop from reserved names.
- [ ] **Step 6: Commit** — `git commit -m "feat: cms_orders with idempotent transition primitive — migration 0011"`

---

### Task 2: `internal/stripe` — client + signature verification

**Files:**
- Create: `internal/stripe/client.go`, `internal/stripe/signature.go`
- Test: `internal/stripe/client_test.go`, `internal/stripe/signature_test.go`

**Interfaces:**
- Produces:
  - `stripe.New(secretKey string, opts ...Option) *Client`; `stripe.WithBaseURL(u string) Option` (test seam — never env); `stripe.WithHTTPClient(c *http.Client) Option`. Internal default: 10 s timeout.
  - `stripe.LineItem{Name string; UnitAmount int64; Currency string; Quantity int}`; `stripe.SessionParams{LineItems []LineItem; SuccessURL, CancelURL string; OrderID string}` — **no client-amount field exists on this type (BR-COM-1 enforcement point)**; `(*Client).CreateCheckoutSession(ctx, p SessionParams) (SessionResult, error)`; `SessionResult{ID, URL string}`. Wire shape: `POST {base}/v1/checkout/sessions`, `Authorization: Bearer <sk>`, form-encoded: `mode=payment`, `success_url`, `cancel_url`, `metadata[order_id]`, and per item `line_items[i][price_data][currency|unit_amount|product_data][name]` + `line_items[i][quantity]`. Non-2xx → `*stripe.APIError{Status int; Code, Message string}` (parsed from Stripe's error JSON; message logged WITHOUT the request body).
  - `stripe.VerifySignature(payload []byte, header, secret string, now time.Time) error` — parse `t=<unix>,v1=<hex>[,v1=…]` (multiple v1 allowed — key rotation); reject `|now-t| > 5m` (`ErrTolerance`); expected = HMAC-SHA256(secret, `<t>.<payload>`); `hmac.Equal` against each v1; none match → `ErrBadSignature`.

- [ ] **Step 1: Failing tests** — client (httptest): golden form-body assertion for a 2-item session (exact key names above), bearer header present, metadata carried, 402 response → `APIError{Status:402}`; signature: valid header verifies; tampered payload fails; `t` 6 min old fails with `ErrTolerance`; second `v1` (rotation) verifies; malformed header fails closed.
- [ ] **Step 2:** FAIL. **Step 3:** implement (≈120 + 60 lines). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: thin Stripe client — checkout sessions + signature verification (D-V3-3)"`

---

### Task 3: Checkout — snapshot, order, session (BR-COM-1)

**Files:**
- Create: `internal/commerce/checkout.go`, `internal/httpapi/checkout_handlers.go` (checkout half)
- Modify: `internal/audit/actions.go` (+`commerce.order.create`), `docs/BUSINESS_RULES.md` (Commerce section, BR-COM-1)
- Test: `internal/commerce/checkout_test.go`

**Interfaces:**
- Consumes: `CartService.Get` (V3-P2), `stripe.Client` (Task 2), sqlc (Task 1).
- Produces:
  - `commerce.NewCheckout(carts *CartService, orders *store.Queries, sc *stripe.Client, rec, log) *Checkout`; `(*Checkout).Start(ctx, userID uuid.UUID, successURL, cancelURL string) (orderID uuid.UUID, url string, err error)` — sequence: load cart (empty → `ErrEmptyCart` → 422); **any stale item → `ErrStaleItems` listing record IDs → 422** (the V3-P2 surface becomes a hard stop); build `line_items` from the **hydrated** cart (`unit_amount` from the live records — the snapshot moment, BR-COM-1), `amount_total = Σ unit_amount × qty`; insert `pending` order (line items, currency, total); `CreateCheckoutSession` with `metadata[order_id]`; success → `SetOrderSession` + audit `commerce.order.create`; Stripe failure → `DeleteOrder` (pending-only) + `ErrGatewayDown` → 503-mapped `unavailable`.
  - `POST /api/v1/checkout {success_url, cancel_url}` (RequireCommerce + end-user gate): URLs must parse absolute http(s) → else 422 naming the field; → `{data:{order_id, url}}`; `no-store`.

- [ ] **Step 1: Failing tests** — happy path against the httptest Stripe stub: order row has line items matching the cart (record_id, title=display, unit_amount, qty) and `amount_total` exact — **`TestBR_COM_1_AmountsSnapshotServerSide`**: change the record's price AFTER checkout → order amounts unchanged; the request body contains no amount field and a client sending one is ignored by shape (handler decodes only the two URL fields — assert unknown-field tolerance doesn't smuggle amounts); stale cart → 422 listing IDs; empty → 422; stub 500 → 503 + pending order deleted; `ftp://` success_url → 422.
- [ ] **Step 2:** FAIL. **Step 3:** implement + BUSINESS_RULES amendment (new "Commerce" section heading, BR-COM-1 text from the spec §D3). `make trace` green (test name traces). **Step 4:** PASS. **Step 5: Commit** — `git commit -m "feat: checkout with server-side price snapshot (BR-COM-1, UAC-3.1 half)"`

---

### Task 4: Webhook + state machine + hooks + erasure (BR-COM-2)

**Files:**
- Create: `internal/commerce/orders.go`
- Modify: `internal/httpapi/checkout_handlers.go` (webhook half), `internal/auth/erasure.go` + its test, `docs/BUSINESS_RULES.md` (BR-COM-2 + BR-AUTH-15 text), `docs/architecture/05-auth-security.md`, V1 secret-log test (+ the two regexes), `internal/audit/actions.go` (+`commerce.order.paid|failed|expired`)
- Test: `internal/commerce/orders_test.go`, handler integration tests

**Interfaces:**
- Produces:
  - `commerce.OrderPaidHook func(ctx context.Context, tx pgx.Tx, order store.CmsOrder) error`; `commerce.NewOrderMachine(pool, q, carts *CartService, rec, log) *OrderMachine`; `(*OrderMachine).SetPaidHook(h OrderPaidHook)` (nil no-op — V3-P4 registers); `(*OrderMachine).Handle(ctx, eventType string, sessionID string) error` — in one tx: resolve order by session (unknown session → logged no-op, 200 — Stripe may replay pre-cutover events); `checkout.session.completed` → `TransitionOrder(pending→paid)`; **0 rows affected → audited no-op (idempotency, D-V3-9)**; 1 row → clear the user's cart + run hook (hook error → rollback, Stripe redelivers) + audit `commerce.order.paid`; `…expired` → `pending→expired`; `…async_payment_failed` → `pending→failed`; unknown event type → 200 ignored.
  - `POST /api/v1/stripe/webhook` — mounted OUTSIDE the end-user gate (Stripe is the caller) but inside `RequireCommerce`; handler: read raw body first (respecting the 64 KiB non-record body cap), `stripe.VerifySignature(body, r.Header.Get("Stripe-Signature"), cfg.Commerce.WebhookSecret, time.Now())` — failure → 400 `validation_failed` "invalid signature" logged WITHOUT the header value (**BR-COM-2**); parse only `{type, data.object.id}`; → `Handle`.
  - Erasure: `auth.EraseEndUser` gains `TombstoneOrdersForUser` inside its transaction; BR-AUTH-15 text amended: "…orders are retained with `end_user_id` rewritten to the tombstone (financial history without identity)"; the BR test gains the assertion.
  - 05 amendment: threat-model row — webhook forgery mitigated by BR-COM-2; replay mitigated by tolerance + idempotent transitions.

- [ ] **Step 1: Failing tests** — **`TestBR_COM_2_SignatureRequiredAndIdempotent`**: unsigned POST → 400, order untouched; correctly signed `completed` → order paid + cart cleared + audit; **same event replayed → 200, no state change, no second audit `paid`, hook NOT re-run**; signed `expired` on the already-paid order → no-op (guard: `pending→expired` affects 0); hook-failure rollback: a hook returning error leaves the order `pending` and the cart intact; unknown session id → 200; secret-log suite green with new regexes; erasure test extended (orders tombstoned, order row count unchanged).
- [ ] **Step 2:** FAIL. **Step 3:** implement + doc amendments. **Step 4:** PASS (`make trace`: BR-COM-2 traces). **Step 5: Commit** — `git commit -m "feat: signature-verified idempotent order webhook + erasure tombstoning (BR-COM-2)"`

---

### Task 5: Admin Orders screen + e2e + manual gate step

**Files:**
- Create: `web/src/routes/Orders.svelte`, `web/e2e/v3p3-orders.spec.ts`
- Modify: `web/src/lib/router.js` (+`#/orders`), nav (admin+), `docs/architecture/04-api-layer.md` (route block)

**Interfaces:**
- Produces: `GET /api/admin/orders?status=&cursor=` (RequireSession + RequireRole(admin), keyset `(created_at,id)` desc) — read-only; no admin mutation of orders exists (state belongs to the webhook stream).

- [ ] **Step 1: Failing e2e** — seed orders via the integration fixtures (one paid, one pending): Orders screen lists both with status chips; status filter narrows; detail drawer shows line items (display, qty, unit amount, total) and the Stripe session id; no edit affordances exist on the screen.
- [ ] **Step 2: Implement** — list + drawer; money rendered from minor units with the order's currency.
- [ ] **Step 3:** e2e PASS.
- [ ] **Step 4: Record the manual gate step** — append to this plan's gate section and the V3-P5 gate checklist: "UAC-3.1 manual verification: with real `sk_test_…` keys, add two products to a cart via the API, `POST /checkout`, complete the Stripe-hosted test checkout with card `4242 4242 4242 4242`, confirm the webhook (Stripe CLI `stripe listen --forward-to`) transitions the order to `paid` with line items matching the cart."
- [ ] **Step 5: Commit** — `git commit -m "feat: read-only admin orders screen (UAC-3.1)"`

---

### Task 6: Acceptance sweep

- [ ] `make test && make trace` green; waiver empty; BR-COM-1/2 trace to their named tests.
- [ ] Full e2e suite green (V1 + V2 + v3p1–p3).
- [ ] `make bench` unchanged.
- [ ] `git grep -rn "sk_test\|sk_live" internal/ web/ --include='*.go' --include='*.js' --include='*.svelte' | grep -v _test` → empty (no hardcoded keys).
- [ ] Doc amendments landed: BUSINESS_RULES (Commerce section + BR-AUTH-15), 04, 05, 07.
- [ ] Manual gate step executed once against Stripe test mode and its outcome recorded in the execution notes (UAC-3.1's literal wording).

## Self-Review Notes (execution-time attention)

- **The `TransitionOrder` zero-rows result is the whole idempotency design** (D-V3-9) — resist adding an event-dedup table; replays and out-of-order deliveries all collapse to no-op transitions, and the test suite pins that.
- **Checkout-failure cleanup deletes only `pending`** orders (`AND status='pending'` in the query) — a race where the webhook lands before the cleanup cannot delete a paid order.
- **`amount_total` recomputation vs Stripe's**: we do not reconcile against the event's `amount_total`; ours is authoritative (BR-COM-1). A mismatch would mean a Stripe-side price edit mid-session — logged at warn if the event carries a different total (cheap check, worth the line).
- **Webhook body cap**: Stripe events for checkout sessions are ~4 KiB; the 64 KiB non-record cap holds — do not raise it.
