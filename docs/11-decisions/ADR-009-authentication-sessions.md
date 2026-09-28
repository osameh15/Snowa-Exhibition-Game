# ADR-009: Authentication and Sessions

Status: **Accepted** (local admin accounts + TOTP for v1 — resolves OQ-21)

## Context
Participants authenticate by phone OTP; admins need strong authentication; displays need restricted read-only access. Clients are browsers (incl. iOS Safari with ITP and standalone PWA); SSE needs auth without custom headers. Participant, admin and display routes share one origin in v1.

## Options
| Option | Assessment |
|---|---|
| JWT in localStorage | XSS-readable; revocation complexity; `EventSource` cannot send headers |
| JWT in cookies | Revocation still needs server state |
| **Server-managed opaque random tokens in HttpOnly cookies, hashed server-side** | Simple revocation, XSS-safe storage, works with SSE |
| SSO for admins | Possible future option |

## Decision
Opaque 256-bit session tokens, stored server-side as SHA-256 hashes with expiry and revocation. Three separate namespaces, each with its own table/middleware and cookie:

| Namespace | Cookie | Attributes | Accepted by |
|---|---|---|---|
| Participant | `sx_ps` | `HttpOnly; Secure; SameSite=Lax; Path=/api/v1` | `/api/v1/*` only |
| Admin | `sx_as` | `HttpOnly; Secure; SameSite=Strict; Path=/api/admin` | `/api/admin/v1/*` only |
| Display | `sx_ds` | `HttpOnly; Secure; SameSite=Strict; Path=/api/display` | `/api/display/v1/*` only (read-only) |

Admins: local named accounts, Argon2id password hashing, mandatory TOTP, lockout, idle/absolute expiry, revocation, step-up TOTP for T4 operations. CSRF: SameSite + required custom header + `Origin` validation on every mutating request. A participant or display session can never authorize an admin API; a display session can never gain admin permissions.

## Consequences
+ Straightforward, revocable, secure defaults; no dependency on network location.
− One indexed DB lookup per request (negligible).
− iOS standalone PWA requires one additional OTP login.
− Shared origin increases XSS blast radius → strict CSP and step-up TOTP (see [Admin architecture §1a](../01-architecture/06-admin-architecture.md#1a-admin-security-baseline-non-negotiable)).
