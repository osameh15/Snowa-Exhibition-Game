# Authentication, OTP and Profile API

Domain: [Participant lifecycle](../02-domain/02-participant-lifecycle.md). Security: [Authentication & sessions](../07-security/02-authentication-and-sessions.md).

## POST `/api/v1/auth/otp/request`

Request (Idempotency-Key required):
```json
{ "phone": "۰۹۱۲ ۳۴۵ ۶۷۸۹" }
```
Server: normalize → `+989123456789`; validate; rate-limit; supersede previous open challenges; create + send.

Response `200`:
```json
{ "challengeId": "uuid", "codeLength": 5, "expiresAt": "…", "resendAvailableAt": "…", "serverTime": "…" }
```
Errors: `400 PHONE_INVALID`, `429 RATE_LIMITED {retryAfterMs}`, `503 OTP_PROVIDER_UNAVAILABLE`.

Enumeration safety: identical response shape and similar latency for new and existing phones. Blocked participants are rejected only at verify.

## POST `/api/v1/auth/otp/verify`

```json
{ "challengeId": "uuid", "code": "۴۷۲۹۱" }
```
Response `200` + `Set-Cookie: sx_ps=…`:
```json
{ "participant": { "id": "uuid", "displayName": null, "profileComplete": false, "consentRequired": false } }
```
Errors: `422 OTP_INVALID {attemptsLeft}`, `422 OTP_LOCKED`, `410 OTP_EXPIRED`, `409 CHALLENGE_USED`, `403 PARTICIPANT_BLOCKED`, `429 RATE_LIMITED`.

On success: existing sessions of the same participant are **not** revoked (multi-device allowed); session rotation happens on every verify.

## POST `/api/v1/auth/logout`
Revokes current session; clears cookie. `204`.

## GET `/api/v1/me`
```json
{ "id": "uuid", "displayName": "علی رضایی", "profileComplete": true, "phoneMasked": "0912•••6789",
  "consentRequired": false, "serverTime": "…" }
```
`phoneMasked` is the participant's own number (they know it) — still masked to limit shoulder-surfing on shared screens.

## PUT `/api/v1/me/profile`
```json
{ "displayName": "علی رضایی", "consentVersion": "2026-10-v1" }
```
- Allowed only while `display_name IS NULL` (v1). Otherwise `409 PROFILE_ALREADY_SET` (until OQ-09 enables editing).
- `consentVersion` required only when `consentRequired`.
- Response `200` → `/me` body. Errors: `400 NAME_INVALID {reason: TOO_SHORT|TOO_LONG|INVALID_CHARS|NO_LETTERS}`.

## GET `/api/v1/me/summary`
Profile page data: per-game best/rank/attempts, tickets (base per game + extra), reward grants (Persian title, code if any, status).
