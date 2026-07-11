# 09 — Deployment

**Version:** 1.1 · **Last Updated:** 2026-07-11 · **Owner:** Miraj Aryal

The deliverable is one statically linked binary embedding the admin UI and the system-table migrations (N-1), targeting linux/amd64 and linux/arm64 (N-2). Deployment is the binary, the environment variables, PostgreSQL 16+, and an S3-compatible bucket — nothing else exists to operate (BR-RUNTIME-2).

## Build

`make build` runs `vite build` in `web/` first, then `go build` with the embedded `web/dist` and `internal/store/migrations` — the Go build fails if either embed target is missing, so a binary with stale or absent UI assets is unrepresentable. Release builds stamp version and commit via `-ldflags`.

## Configuration

All configuration enters through the environment variables of `../BUSINESS_RULES.md` § Naming Constants — the sixteen-variable table is exhaustive. The binary validates required variables at startup and exits non-zero listing every missing one at once, not the first.

## Startup (BR-RUNTIME-3)

1. Validate configuration; open the database pool.
2. Run embedded migrations under a Postgres advisory lock — a concurrently restarting replacement process waits rather than racing.
3. Acquire the process-lifetime instance lock (BR-RUNTIME-8): a session-scoped `pg_advisory_lock` on a dedicated key, distinct from the migration and schema keys. A second process fails startup immediately with a clear log line — the migration lock is something a replacement process waits on, but the instance lock is not: stop-then-start remains the only supported upgrade path. *(Resolves EC-16.)*
4. Load the schema cache.
5. When `cms_users` is empty, generate and log the single-use setup token and enable `/setup` (BR-AUTH-11); when `CMS_RECOVERY_EMAIL` names an existing user, generate and log the single-use recovery token and enable `/recover` (BR-AUTH-12). Both tokens are logged once at `warn`, carry a 30-minute TTL, and die on use or process exit.
6. Open the HTTP listener on `CMS_PORT`; `/readyz` begins returning 200.
7. The publisher's first tick, immediately after the listener opens, runs the scheduled-publish catch-up scan: publish every record whose `publish_at` elapsed while the binary was down, logging each late publication (BR-LIFE-9; `08-observability.md`). *(Resolves EC-13, deployment half.)*

Migration failure, instance-lock failure (BR-RUNTIME-8), key-load failure (BR-AUTH-10), or schema-cache failure aborts startup with a non-zero exit — the system fails closed (N-11) rather than serving with partial state.

## Shutdown (EC-14)

On SIGTERM/SIGINT the binary: (1) flips `/readyz` and `/healthz` to 503 so the proxy stops routing new work — proxies watch `/readyz`, supervisors watch `/healthz` (`08-observability.md`), (2) stops the job tickers, (3) drains in-flight requests via `http.Server.Shutdown` with the 15-second window (BR-RUNTIME-6), (4) exits 0. Requests accepted before the signal complete; requests arriving after it receive the proxy's upstream error, not a dropped connection. A drain exceeding 15 seconds forcibly closes and exits 1 — visible in logs and exit status. *(Resolves EC-14.)*

**Upgrade procedure:** single tenant, single process — stop, replace binary, start. The window is bounded by the drain plus startup (≤ 2 s when migrations are current, N-5). Zero-downtime orchestration is deliberately out of architectural scope; the health endpoints make any supervisor (systemd `Restart=always` with `TimeoutStopSec=20`, or a container runtime honoring SIGTERM) sufficient. This single-instance posture is deliberate (BR-RUNTIME-8): every upgrade, and every unplanned crash-restart, costs a drain-plus-startup gap rather than failing over to a peer. Target availability class is **~99.5%**; high availability is explicitly out of scope (N-13).

## Timeouts

`http.Server` and query-level timeouts are compiled-in constants, not environment variables — they do not appear in the sixteen-variable table above.

| Setting | Value |
|---|---|
| `http.Server` `ReadHeaderTimeout` | 5 s |
| `http.Server` `ReadTimeout` | 30 s |
| `http.Server` `WriteTimeout` | 30 s |
| `http.Server` `IdleTimeout` | 120 s |
| Per-request context deadline | 25 s |
| Postgres `statement_timeout` — collection queries | 10 s |
| Postgres `statement_timeout` — schema transactions | 60 s |
| Request body cap — record-write routes (`.../records` create/update) | 5 MiB |
| Request body cap — all other routes (auth, presign, finalize, etc.) | 64 KiB |

The 5 MiB record-write cap is sized to accommodate the 1 MiB per-field cap (`02-core-interfaces.md` `Document.Set`) with headroom for multi-field records; the 64 KiB cap covers every non-content route (`04-api-layer.md`).

## Reverse-Proxy Contract

The binary expects an edge proxy (Cloudflare or equivalent) in front of it. The contract has four hard requirements:

1. **TLS terminates at the edge**; the session cookie carries `Secure` and never transits plaintext (BR-AUTH-1).
2. **The proxy appends to `X-Forwarded-For`** — never forwards the client-supplied header untouched. List your edge's egress ranges in `CMS_TRUSTED_PROXY_CIDRS` (comma-separated CIDRs — e.g., Cloudflare's published IP ranges, refreshed occasionally as Cloudflare updates them); an empty value falls back to the built-in loopback/RFC1918 heuristic only. An unlisted proxy is not trusted by omission: the rate limiter degrades to per-proxy-IP limiting rather than failing open. The rate limiter's deterministic trust rule depends on this (`05-auth-security.md` §5, EC-10).
3. **The proxy respects origin cache headers** — no cache-everything overrides — or the SPA contract below breaks.
4. **The proxy honors origin cache headers on `/api/*` too.** Public API responses carry `Cache-Control: public, s-maxage=60, stale-while-revalidate=60` (anonymous) or `no-store` (any request bearing `Authorization` or a cookie) per BR-API-5 (`04-api-layer.md`); an edge override that ignores these headers risks caching a credentialed response for the wrong caller.

## SPA Cache Busting (EC-15)

- Hashed assets (`/assets/app.3f9c2b7e.js`) serve with `Cache-Control: public, max-age=31536000, immutable` — safe forever because a content change changes the name.
- `index.html` serves with `Cache-Control: no-cache`: browsers and CDN revalidate it on every navigation, so the first request after a binary upgrade fetches the new `index.html` referencing the new hashes.
- The failure mode this kills: a cached `index.html` pointing at purged asset hashes. Because `index.html` is never cached without revalidation and old assets die only when the binary is replaced (releasing new hashes atomically with the new index), the stale-reference window is a single in-flight navigation, which a reload resolves. *(Resolves EC-15.)*

## Media Origin & CORS

The media bucket needs a CORS policy and an origin that is isolated from the admin UI:

- **Bucket CORS policy:** allow `PUT` from the admin origin (and any registered app origins that upload directly); never wildcard the allowed origin on credentialed requests. Without this, the browser blocks the direct-to-storage `PUT` that the presign flow depends on — this is a prerequisite for UAC-1.5 (presign → direct upload → finalize), not an optional hardening step.
- **Origin isolation:** `R2_PUBLIC_BUCKET_URL` must not share a registrable domain with the admin UI origin. This is the sentence `06-admin-ui.md`'s CSP section cites back to this document: if the media origin shared a registrable domain with the admin origin, a stored HTML file in the media bucket could execute with the admin origin's privileges (stored-HTML/phishing containment, `05-auth-security.md` §6).

## Backup and Restore

- **Database — PITR required.** Point-in-time recovery via either managed Postgres with PITR enabled, or self-hosted WAL archiving (pgBackRest or WAL-G), operated per the restore-drill runbook below. Targets: **RPO ≤ 5 minutes, RTO ≤ 1 hour** (N-12).
- Scheduled `pg_dump` remains the portable second copy — the schema uses no non-bundled extensions (N-9), so any PostgreSQL 16+ restores it. `pg_dump` complements PITR (environment portability, an independent copy) but does not by itself meet the RPO/RTO targets.
- **Object storage:** bucket versioning covers media; `cms_media.object_key` values are the join between a database restore and the bucket.
- **Restore order:** database first (from PITR or the latest `pg_dump`, depending on the scenario), then verify bucket reachability, then start the binary — startup migration idempotency tolerates a dump taken mid-version.
- A restore drill is required before the V1 release (N-12; see Runbooks below).
- Schema export/import (V2, F-25) complements this for environment promotion; it does not replace PITR/`pg_dump` for disaster recovery.

## Runbooks

**JWT key rotation** (BR-AUTH-10; mechanics in `05-auth-security.md` §3): generate a successor RSA-2048 keypair with a new `kid` → begin issuing with the new key while continuing to verify tokens against both `kid`s → retire the old key row once 15 minutes (the maximum JWT TTL) have passed, so no live token can still reference it. **Master-secret-loss recovery:** if `CMS_MASTER_SECRET` is lost, regenerate the keypair; outstanding access JWTs (≤15 minutes old) fail verification, and clients silently re-issue via their refresh tokens — opaque hashed rows independent of the RSA key — so no re-login is required (`05-auth-security.md` §3).

**Super-admin recovery** (BR-AUTH-12): set `CMS_RECOVERY_EMAIL` to an existing admin's email address and restart the binary. Startup step 5 above logs the single-use recovery token once at `warn`; visit `/recover` and consume the token within 30 minutes to set a new password and revoke that user's sessions. Unset the variable again afterward — it is a one-time gate, not standing configuration.

**Restore drill** (N-12, required before V1 ships): provision a scratch Postgres instance, restore the most recent PITR base-plus-WAL (or the latest `pg_dump`) into it, point a binary at it, and confirm collection data and system tables are intact and the schema cache loads cleanly. Record the wall-clock time from "declare an outage" to "binary serving traffic again" against the RTO ≤ 1 hour target, and the data-loss window against the RPO ≤ 5 minute target. Re-run whenever the backup mechanism or provider changes.

**Hot-DDL guidance** (`03-dynamic-schema.md`): field type changes and V2 FTS `tsvector` rebuilds hold `ACCESS EXCLUSIVE` on the target table for the rewrite's duration — reads stall, not just writes. Run these off-peak; `schema.Engine.Apply` logs each operation's duration (`08-observability.md`), so operators can see the actual stall window rather than guess at it.

## Edge-Case Coverage (this document)

| EC | Resolution |
|---|---|
| EC-13 | Publisher's first tick, immediately after the listener opens, runs the catch-up scan (Startup) |
| EC-14 | Readyz/healthz-flip → ticker stop → 15 s drain → bounded force-close (Shutdown) |
| EC-15 | Immutable hashed assets + no-cache index + proxy header contract (SPA Cache Busting) |
| EC-16 | Process-lifetime instance lock; a second process fails startup with a clear log line (Startup) |
