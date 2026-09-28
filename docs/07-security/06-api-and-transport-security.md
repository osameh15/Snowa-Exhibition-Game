# API and Transport Security

| Area | Requirement |
|---|---|
| Transport | TLS 1.2+ (1.3 preferred), HSTS, HTTP→HTTPS redirect; certificates auto-renewed (ACME) where the provider allows |
| Input validation | zod schemas at every boundary (shared `contracts`); reject unknown fields on admin/config endpoints; string length limits; Unicode normalization for names and admin text |
| Output encoding | Vue template escaping; no `v-html` with user/admin content; admin-authored Persian text rendered as text only |
| AuthZ | Deny by default; per-route permission declaration checked at startup (routes without a declared policy fail boot) |
| Object access | Participant endpoints derive identity from session; foreign ids → 404 |
| Mass assignment | Explicit DTO → domain mapping; never spread request bodies into DB rows |
| SQL | Parameterized queries only (query builder/ORM); raw SQL reviewed |
| Errors | No stack traces or SQL errors in responses; `requestId` for support |
| Headers | CSP (self-only), `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` (camera/mic/geolocation off), `frame-ancestors 'none'` |
| CORS | Not enabled (same-origin by proxy) |
| Secrets | Env/secret store only; separate per environment; rotated before event; CI secret scanning |
| Dependencies | Lockfile, automated vulnerability scan in CI, minimal dependency policy |
| Outbound | Worker egress restricted to SMS and Snowa hosts where infrastructure allows |
| Health endpoints | Expose no data; `/metrics` internal only |
| File uploads | Only admin CSV (codes): size ≤ 5 MB, parsed as text, never stored as executable/served back |
| Logging | Structured, redacted (see [Observability](../09-operations/03-observability.md)) |

## Security audit requirements

What MUST produce audit records is defined in [Audit model](../02-domain/08-audit-model.md). Security-relevant additions:
- All admin authentication events (success/failure/lockout/MFA failure).
- Full phone reveals and exports (who, what filter, row count).
- Rate-limit lockouts for admin accounts.
- Integrity reconciliation mismatches.
- Display token creation/revocation.

Alerting on: repeated admin login failures, T4 actions outside event hours, bulk exports, audit hash-chain verification failure.
