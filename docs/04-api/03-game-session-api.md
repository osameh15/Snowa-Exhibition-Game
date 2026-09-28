# Game Session and Score Submission API

Domain: [Game/session/attempt lifecycle](../02-domain/03-game-session-attempt-lifecycle.md). Validation: [Score validation framework](../05-games/02-score-validation-framework.md).

## POST `/api/v1/game-sessions`

Headers: `Idempotency-Key` (required).
```json
{ "game": "spin-perfect", "runtimeVersion": "1.3.0", "client": { "viewport": "390x844", "dpr": 3, "reducedMotion": false } }
```
Checks: authenticated, profile complete, consent (if required), event LIVE, game ENABLED, `attempts_used < allowed`, no STARTED session before `late_deadline_at`, runtime ≥ `runtime_min_version`.

`201` (new) / `200` (existing ISSUED returned):
```json
{
  "sessionId": "uuid",
  "state": "ISSUED",
  "game": "spin-perfect",
  "configVersion": "1.0.0",
  "configChecksum": "sha256-hex",
  "seed": "32-hex-chars",
  "params": { "...": "game-specific tuning, see 05-games" },
  "durationMs": 40000,
  "pauseBudgetMs": 60000,
  "startBy": "…",
  "attemptsUsed": 0,
  "attemptsAllowed": 1,
  "serverTime": "…"
}
```
Errors: `409 GAME_UNAVAILABLE`, `409 EVENT_NOT_LIVE`, `409 ATTEMPTS_EXHAUSTED`, `409 SESSION_ALREADY_ACTIVE {sessionId, state}`, `426 CLIENT_UPGRADE_REQUIRED`, `403 PROFILE_INCOMPLETE`.

## POST `/api/v1/game-sessions/{id}/start`

Headers: `Idempotency-Key` (required). Empty body.
`200`:
```json
{ "sessionId": "uuid", "state": "STARTED", "attemptNumber": 2, "startedAt": "…", "deadlineAt": "…", "serverTime": "…" }
```
Repeated call on a STARTED session returns the same body. Errors: `409 GAME_UNAVAILABLE` (session CANCELLED, no attempt consumed), `409 ATTEMPTS_EXHAUSTED`, `409 SESSION_EXPIRED`, `404`.

## POST `/api/v1/game-sessions/{id}/result`

Idempotency is natural: one result per session. The client still sends `Idempotency-Key: <sessionId>` for uniform handling.

```json
{
  "runtimeVersion": "1.3.0",
  "configVersion": "1.0.0",
  "configChecksum": "sha256-hex",
  "claimedScore": 8750,
  "activeMs": 40000,
  "pausedMs": 3200,
  "endReason": "TIMER",
  "stats": { "hits": 31, "perfects": 18, "maxCombo": 11 },
  "actions": [[412, "tap"], [1180, "tap"], [1937, "tap"]],
  "pauses": [[15320, 3200, "hidden"]]
}
```

| Field | Rule |
|---|---|
| `actions` | Array of compact tuples `[t_ms, type, ...args]`, `t_ms` integer gameplay time, non-decreasing, `0 ≤ t ≤ activeMs`. Per-game types in the game specs. Max entries from `bounds.max_actions` |
| `pauses` | `[t_ms_at_pause, duration_ms, reason]`; Σ duration = `pausedMs` ≤ `pauseBudgetMs` |
| `endReason` | `TIMER` (normal) · `PAUSE_BUDGET_EXHAUSTED` · `USER_QUIT` (explicit leave with confirm — still submitted, scored as-is) · `CONTEXT_LOST` (WebGL context not recoverable — scored as-is) |
| `stats` | Client-computed; compared to server recomputation; not authoritative |

`200` — authoritative ResultModel [SPEC §8.3]:
```json
{
  "sessionId": "uuid",
  "attemptId": "uuid",
  "status": "ACCEPTED",
  "attemptScore": 8750,
  "bestScore": 8750,
  "previousBestScore": 6420,
  "isNewBest": true,
  "rank": 42,
  "baseTicket": { "grantedNow": true, "active": true },
  "extraRewards": [
    { "grantId": "uuid", "type": "DISCOUNT_CODE", "titleFa": "…", "descriptionFa": "…", "code": "SNOWA20", "validUntil": "…" }
  ],
  "attemptsUsed": 1,
  "attemptsAllowed": 3,
  "canReplay": true,
  "rewardsPending": false
}
```
- `status: "REJECTED"` responses are `200` with `attemptScore: null`, `rejectReason` code, `bestScore` unchanged, `baseTicket.grantedNow: false` — the UI shows a neutral Persian "result could not be verified" message and the lobby. (Rejected is a business outcome, not a transport error.)
- `ACCEPTED_FLAGGED` is reported to the participant as `ACCEPTED` (flags are internal).
- `rewardsPending: true` when reward evaluation was deferred.

Errors: `409 RESULT_ALREADY_SUBMITTED` (different payload), `409 SESSION_NOT_STARTED`, `409 SESSION_EXPIRED` (after `late_deadline_at`), `400 VALIDATION_FAILED` (malformed; the attempt is **not** consumed further and the client may fix and resend only if the session is still open — in practice a client bug), `503 RETRYABLE`.

Duplicate submission (same payload hash) → `200` with the identical stored ResultModel (AC-019).

## GET `/api/v1/game-sessions/{id}`

For refresh recovery.
```json
{ "sessionId": "uuid", "state": "SUBMITTED", "game": "spin-perfect", "attemptNumber": 1,
  "deadlineAt": "…", "result": { "...ResultModel": "..." } }
```
`result` present only when SUBMITTED. Rank is recomputed at read time.

## POST `/api/v1/game-sessions/{id}/abandon`

Optional explicit abandonment (participant confirmed "leave game"). ISSUED → CANCELLED (no attempt). STARTED → attempt ABANDONED immediately (frees the participant to start the next attempt without waiting for the deadline). `204`.

## Timing and clocks

- Server authority for timing: `started_at` (DB time at `/start`) and receive time of `/result`.
- `server_elapsed_ms = received_at − started_at` MUST satisfy `server_elapsed_ms ≥ activeMs + pausedMs − tolerance` (tolerance default 1 500 ms, covers client start latency jitter) — otherwise `REJECTED: TIMING_IMPOSSIBLE`.
- `activeMs` MUST equal `durationMs` (± 100 ms) for `endReason = TIMER`; for `PAUSE_BUDGET_EXHAUSTED`, `pausedMs ≥ pauseBudgetMs − 100`; for `USER_QUIT`/`CONTEXT_LOST`, `activeMs ≤ durationMs`.
- A shortened attempt can only score less than a full one (scoring is monotonic in time), so early endings carry no advantage.
- Received after `deadline_at` but before `late_deadline_at` → accepted with flag `LATE_SUBMISSION` (covers network outages; no competitive advantage since the log is validated the same way).
