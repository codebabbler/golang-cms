# 09 — Deployment

**Version:** 1.0 · **Last Updated:** 2026-07-08 · **Owner:** Miraj Aryal

The deliverable is one statically linked binary embedding the admin UI and the system-table migrations (N-1), targeting linux/amd64 and linux/arm64 (N-2). Deployment is the binary, the environment variables, PostgreSQL 16+, and an S3-compatible bucket — nothing else exists to operate (BR-RUNTIME-2).

## Build

`make build` runs `vite build` in `web/` first, then `go build` with the embedded `web/dist` and `internal/store/migrations` — the Go build fails if either embed target is missing, so a binary with stale or absent UI assets is unrepresentable. Release builds stamp version and commit via `-ldflags`.

## Configuration

All configuration enters through the environment variables of `../BUSINESS_RULES.md` § Naming Constants — the fourteen-variable table is exhaustive. The binary validates required variables at startup and exits non-zero listing every missing one at once, not the first.

## Startup (BR-RUNTIME-3)

1. Validate configuration; open the database pool.
2. Run embedded migrations under a Postgres advisory lock — a concurrently restarting replacement process waits rather than racing.
3. Load the schema cache.
4. When `cms_users` is empty, generate and log the single-use setup token and enable `/setup` (BR-AUTH-11).
5. Run the scheduled-publish catch-up scan: publish every record whose `publish_at` elapsed while the binary was down, logging each late publication (BR-LIFE-9; `08-observability.md`). *(Resolves EC-13, deployment half.)*
6. Open the HTTP listener on `CMS_PORT`; `/healthz` begins returning 200.

Migration failure, key-load failure (BR-AUTH-10), or schema-cache failure aborts startup with a non-zero exit — the system fails closed (N-11) rather than serving with partial state.

## Shutdown (EC-14)

On SIGTERM/SIGINT the binary: (1) flips `/healthz` to 503 so the proxy stops routing new work, (2) stops the job tickers, (3) drains in-flight requests via `http.Server.Shutdown` with the 15-second window (BR-RUNTIME-6), (4) exits 0. Requests accepted before the signal complete; requests arriving after it receive the proxy's upstream error, not a dropped connection. A drain exceeding 15 seconds forcibly closes and exits 1 — visible in logs and exit status. *(Resolves EC-14.)*

**Upgrade procedure:** single tenant, single process — stop, replace binary, start. The window is bounded by the drain plus startup (≤ 2 s when migrations are current, N-5). Zero-downtime orchestration is deliberately out of architectural scope; the health endpoint makes any supervisor (systemd `Restart=always` with `TimeoutStopSec=20`, or a container runtime honoring SIGTERM) sufficient.

## Reverse-Proxy Contract

The binary expects an edge proxy (Cloudflare or equivalent) in front of it. The contract has three hard requirements:

1. **TLS terminates at the edge**; the session cookie carries `Secure` and never transits plaintext (BR-AUTH-1).
2. **The proxy appends to `X-Forwarded-For`** — never forwards the client-supplied header untouched. The rate limiter's deterministic trust rule depends on this (`05-auth-security.md` §5, EC-10).
3. **The proxy respects origin cache headers** — no cache-everything overrides — or the SPA contract below breaks.

## SPA Cache Busting (EC-15)

- Hashed assets (`/assets/app.3f9c2b7e.js`) serve with `Cache-Control: public, max-age=31536000, immutable` — safe forever because a content change changes the name.
- `index.html` serves with `Cache-Control: no-cache`: browsers and CDN revalidate it on every navigation, so the first request after a binary upgrade fetches the new `index.html` referencing the new hashes.
- The failure mode this kills: a cached `index.html` pointing at purged asset hashes. Because `index.html` is never cached without revalidation and old assets die only when the binary is replaced (releasing new hashes atomically with the new index), the stale-reference window is a single in-flight navigation, which a reload resolves. *(Resolves EC-15.)*

## Backup and Restore

- **Database:** plain `pg_dump` on schedule; the schema uses no non-bundled extensions (N-9), so any PostgreSQL 16+ restores it.
- **Object storage:** bucket versioning covers media; `cms_media.object_key` values are the join between a database restore and the bucket.
- **Restore order:** database first, then verify bucket reachability, then start the binary — startup migration idempotency tolerates a dump taken mid-version.
- Schema export/import (V2, F-25) complements this for environment promotion; it does not replace `pg_dump` for disaster recovery.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-13 | Startup catch-up scan before the listener opens (Startup) |
| EC-14 | Health-flip → ticker stop → 15 s drain → bounded force-close (Shutdown) |
| EC-15 | Immutable hashed assets + no-cache index + proxy header contract (SPA Cache Busting) |
