# Internal API Principles

Applies to all platform APIs. Machine-readable specification: OpenAPI 3.1 generated from `packages/contracts` zod schemas in Phase 1 (the documents in this folder are the normative contract until then).

## 1. Namespaces

| Prefix | Consumer | Auth |
|---|---|---|
| `/api/v1/*` | Participant app | participant session cookie `sx_ps` (except auth + analytics) |
| `/api/admin/v1/*` | Admin app | admin session cookie `sx_as` + RBAC |
| `/api/display/v1/*` | Public display | display cookie `sx_ds` |
| `/healthz`, `/readyz` | Infrastructure | none (no data) |
| `/metrics` | Monitoring | internal network only |

## 2. Conventions

| Topic | Rule |
|---|---|
| Format | JSON UTF-8; `camelCase` fields in API; `snake_case` in DB |
| Ids | UUID strings; games addressed by `slug` |
| Time | ISO 8601 UTC strings (`2026-10-01T09:30:12.345Z`); durations as integer ms (`durationMs`) |
| Numbers | Integers for scores/counts; never localized in API |
| Server time | Responses touching timers include `serverTime` so clients can compute offsets |
| Pagination | Keyset: `?limit=&cursor=`; response `{ items, nextCursor }` |
| Versioning | Path major version; additive changes only within a version; clients ignore unknown fields |
| Caching | `Cache-Control: no-store` on all authenticated responses; static leaderboard for display `no-store` too (freshness handled by SSE) |
| Compression | gzip/br at proxy |
| Request limits | Default body 16 KB; result submission 128 KB; analytics 64 KB |

## 3. Idempotency

Mutating endpoints accept `Idempotency-Key: <uuid>` (required where noted). Semantics in [Invariants §3](../03-data/03-invariants-and-transactions.md#3-idempotency-keys). Clients MUST reuse the key when retrying the same intent and MUST generate a new key for a new intent.

## 4. Error model

```json
{
  "error": {
    "code": "ATTEMPTS_EXHAUSTED",
    "retryable": false,
    "retryAfterMs": null,
    "requestId": "01J…",
    "details": { "attemptsUsed": 1, "attemptsAllowed": 1 }
  }
}
```

No human-readable message is shown to participants; the client maps `code` → Persian copy.

| HTTP | Codes |
|---|---|
| 400 | `VALIDATION_FAILED` (details: field paths), `PHONE_INVALID`, `NAME_INVALID` |
| 401 | `UNAUTHENTICATED`, `SESSION_EXPIRED` |
| 403 | `FORBIDDEN`, `PROFILE_INCOMPLETE`, `PARTICIPANT_BLOCKED`, `CONSENT_REQUIRED`, `MFA_REQUIRED`, `STEP_UP_REQUIRED` |
| 404 | `NOT_FOUND` (also used for other participants' resources — no existence leak) |
| 409 | `GAME_UNAVAILABLE`, `EVENT_NOT_LIVE`, `ATTEMPTS_EXHAUSTED`, `SESSION_ALREADY_ACTIVE`, `SESSION_NOT_STARTED`, `SESSION_EXPIRED`, `RESULT_ALREADY_SUBMITTED`, `CHALLENGE_USED`, `VERSION_CONFLICT`, `REQUEST_IN_PROGRESS`, `DRAW_STATE_INVALID` |
| 410 | `OTP_EXPIRED` |
| 422 | `OTP_INVALID` (details: `attemptsLeft`), `OTP_LOCKED`, `IDEMPOTENCY_KEY_REUSED`, `NOT_ENOUGH_ELIGIBLE`, `RULE_UNSAFE` |
| 426 | `CLIENT_UPGRADE_REQUIRED` (runtime older than `runtime_min_version`) |
| 429 | `RATE_LIMITED` (+ `retryAfterMs`) |
| 503 | `RETRYABLE` (transient DB/dependency), `OTP_PROVIDER_UNAVAILABLE` |

## 5. Security headers and transport

HTTPS only (HSTS), `Content-Security-Policy` with self-only sources (+ `blob:` for Phaser workers if needed), `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal. Cookies (one origin, three namespaces — [ADR-009](../11-decisions/ADR-009-authentication-sessions.md)): participant `sx_ps` (`SameSite=Lax; Path=/api/v1`), admin `sx_as` (`SameSite=Strict; Path=/api/admin`), display `sx_ds` (`SameSite=Strict; Path=/api/display`), all `HttpOnly; Secure`. Each namespace's middleware accepts only its own cookie. CSRF: SameSite + required custom header `X-Requested-With: sx` on all mutating requests + `Origin` must equal the deployment origin. `/admin/*` and `/display/*` responses add `X-Robots-Tag: noindex, nofollow, noarchive`.

## 6. Authorization rule

Every handler resolves the actor and checks: (1) authentication, (2) resource ownership (participant) or permission (admin), (3) state preconditions. Participant endpoints never accept a `participantId` parameter — the participant is always derived from the session.
