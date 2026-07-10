# 11 — Roadmap

**Version:** 1.0 · **Last Updated:** 2026-07-07 · **Owner:** Miraj Aryal

The system ships in three releases. The scopes below are authoritative and match the generation contract verbatim; acceptance for each version is defined by `../REQUIREMENTS.md` §6 (UAC-1.x through UAC-3.x). Anything in `../REQUIREMENTS.md` §5 (O-1…O-12) stays out of every release.

## Version Scopes

- **Version 1 (MVP):** Config-driven dynamic schema/CRUD on real tables via the runtime-DDL engine; full content versioning (revisions, pending drafts of published records, restore and compare); soft delete (trash); optimistic locking; Auth MVP (admin sessions, API keys, public JWTs with refresh-token reuse detection); presigned direct-to-R2 uploads; embedded Svelte admin UI via `go:embed`; Tiptap JSONB integration; capped limit/offset pagination; embedded startup migrations; audit event interface with log-only sink.
- **Version 2 (Polish & Search):** SEO meta injection; 301/302 redirects; Postgres FTS including the Tiptap plaintext walker; `?format=html` rich-text rendering; webhooks; complex UI relationships (many-to-many); audit log storage and admin UI; import/export (schema import/export and content export first-class; content import best-effort with documented limitations); scheduled publishing; keyset cursor pagination on the public API.
- **Version 3 (Commerce & Video):** E-commerce cart state; Stripe integration; video access paywalls with delegated transcoding and signed playback tokens; custom Tiptap blocks (Products/Videos).

## Sequencing Rationale

**V1 builds the load-bearing walls.** The schema engine, the live-table/revisions contract, the trash filter, optimistic locking, and the audit event interface are invariants that every later feature assumes; retrofitting any of them would force a rewrite of every query path. V1 therefore carries them even where a leaner MVP could defer them (BR-LIFE-1, BR-LIFE-4, BR-LIFE-7, BR-AUDIT-1).

**V2 spends what V1 saved.** Full-text search reads real columns provisioned in V1; the audit UI persists events whose call sites V1 already wired (BR-AUDIT-2); scheduled publishing reuses the V1 job scheduler; import/export serializes schema metadata that has been JSONB-shaped since V1. No V2 feature requires reopening a V1 contract.

**V3 is additive commerce.** Carts, orders, and paywalls are system tables and handlers on top of the finished platform; the only cross-cutting V3 item is custom Tiptap blocks, which extend the V1 JSONB document format without altering stored documents.

## Delivery Gates

| Version | Gate |
|---|---|
| V1 | UAC-1.1 … UAC-1.5 pass end-to-end; CI `trace` job green over all V1-tagged BR rules (N-10). |
| V2 | UAC-2.1 … UAC-2.4 pass end-to-end; V1 gates stay green. |
| V3 | UAC-3.1 … UAC-3.3 pass end-to-end; V1 and V2 gates stay green. |

## Post-V3 Candidates (Not Committed)

Localization (O-6) is the known major-version undertaking should it ever be needed. No other post-V3 work is planned; candidates enter this document only through a new requirements cycle.
