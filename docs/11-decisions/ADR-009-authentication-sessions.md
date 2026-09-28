# ADR-009: Authentication and Sessions

Status: **Proposed** (admin identity method pending OQ-21)

## Context
Participants authenticate by phone OTP; admins need strong authentication; displays need restricted read-only access. Clients are browsers (incl. iOS Safari with ITP and standalone PWA), SSE needs auth without custom headers.

## Options
| Option | Assessment |
|---|---|
| JWT access/refresh in localStorage | XSS-readable, revocation complexity, `EventSource` cannot send headers |
| JWT in cookies | Revocation still needs server state; no real benefit here |
| **Opaque random session tokens in HttpOnly cookies, hashed server-side** | Simple revocation, XSS-safe storage, works with SSE, same-origin |
| Third-party auth (SSO) for admins | Good if Snowa has an IdP (OQ-21) |

## Decision
Opaque 256-bit session tokens in `HttpOnly; Secure; SameSite=Lax` cookies, stored as SHA-256 hashes with expiry and revocation. Separate cookies/origins for participant, admin and display. Admins: local accounts with argon2id + mandatory TOTP (replaceable by SSO later). CSRF: SameSite + custom header + Origin check.

## Consequences
+ Straightforward, revocable, secure defaults.
− One DB lookup per request (indexed; negligible). Session cache possible later.
− iOS standalone PWA requires one additional OTP login.
