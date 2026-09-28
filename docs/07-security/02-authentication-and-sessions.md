# Authentication, OTP Abuse Protection and Session Security

Decisions: [ADR-009](../11-decisions/ADR-009-authentication-sessions.md), [ADR-005](../11-decisions/ADR-005-otp-provider-boundary.md).

## 1. Participant authentication

| Aspect | Design |
|---|---|
| Factor | Possession of phone (SMS OTP) [SPEC PD-02] |
| Code | 5 digits (OQ-12), CSPRNG, TTL 120 s, HMAC-hashed at rest, constant-time compare |
| Verify attempts | 5 per challenge → LOCKED; only latest challenge valid |
| Session token | 256-bit random, sent as `sx_ps` cookie (`HttpOnly; Secure; SameSite=Lax; Path=/`), stored hashed server-side |
| Session lifetime | absolute: event end + 1 day, max 30 days (A-11); no sliding extension beyond that |
| Revocation | logout, participant block, admin "revoke sessions" |
| Multi-device | allowed (phone + tablet); game session concurrency still enforced per participant+game |
| Session fixation | new token on each verify |
| iOS standalone PWA | separate cookie jar → one extra OTP login (documented) |

Why cookies rather than bearer tokens in `localStorage`: immune to token theft via XSS reads, automatic on SSE (`EventSource` cannot set headers), simpler same-origin model.

## 2. OTP abuse protection

| Control | Default | Notes |
|---|---|---|
| Phone validation | Iranian mobile `+989XXXXXXXXX` only | Blocks premium/international SMS pumping |
| Resend cooldown | 60 s | UI countdown |
| Per phone | 5 sends / 15 min; 10 / 24 h | 429 with `retryAfterMs` |
| Per IP class | 60 requests / 10 min per /24 (IPv4) or /56 (IPv6) | **Generous**: venue Wi-Fi/carrier NAT puts many visitors behind one IP. Tunable live |
| Venue allowlist | Optional higher limits for known venue Wi-Fi egress IPs | Ops config |
| Verify attempts | 5 per challenge; 20 failed verifies per phone per 24 h → 1 h block | |
| Global SMS budget | circuit breaker at N sends/min (config, e.g. 3× expected peak) → alert; optional step-up challenge | Cost protection |
| Step-up challenge | Feature flag: add a lightweight self-hosted challenge (e.g., proof-of-work or simple captcha) only under attack | Must be self-hosted (A-05) |
| Enumeration | Same response and similar timing for all valid phones | |
| Logging | Never log codes; phones masked | |

## 3. Admin authentication

| Aspect | Design |
|---|---|
| Accounts | Named local accounts (v1; SSO possible later) |
| Password | ≥ 12 chars, argon2id (memory ≥ 64 MB, t=3), breached-password check list (offline) |
| MFA | TOTP (RFC 6238), mandatory; recovery codes (Super Admin reset) |
| Lockout | 5 failures / 15 min; alerts |
| Session | `sx_as` cookie (`HttpOnly; Secure; SameSite=Strict; Path=/api/admin`), separate namespace from participant/display, idle 30 min, absolute 12 h; step-up TOTP (≤ 5 min old) for T4 actions |
| Network | Optional defense in depth only: reverse-proxy IP allowlist, VPN, extra proxy auth on staging; admin security never depends on network location |
| Crawlers | `/admin/*`, `/display/*`: `X-Robots-Tag: noindex, nofollow, noarchive` |

## 4. Display authentication

Random 256-bit token (shown once as QR/URL) → exchanged at `/api/display/v1/auth` for `sx_ds` cookie; token hash stored; revocation immediate; scope read-only display APIs.

## 5. Session security checklist

- HTTPS everywhere, HSTS (preload after domain confirmation).
- Cookies never accessible to JS; no tokens in URLs (except one-time display token, immediately exchanged, `Referrer-Policy` prevents leakage).
- CSRF: SameSite=Lax + `X-Requested-With` header + Origin check.
- CSP: `default-src 'self'; script-src 'self'; connect-src 'self'; img-src 'self' data: blob:; media-src 'self' blob:; worker-src 'self' blob:; frame-ancestors 'none'`.
- Rate limit authenticated endpoints per session as well (see [Rate limiting](04-rate-limiting-and-abuse.md)).
