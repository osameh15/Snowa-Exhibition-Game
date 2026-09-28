# Game Availability, Game Session and Attempt Lifecycle

Source: SPEC §7, §8, §16.2, §21, §22.2, PD-05, PD-09, AC-005, AC-006, AC-019. API: [Game session API](../04-api/03-game-session-api.md). Runtime: [Shared game runtime](../05-games/01-shared-game-runtime.md).

## 1. Game availability

```mermaid
stateDiagram-v2
  [*] --> DISABLED: seeded
  DISABLED --> ENABLED: admin enable
  ENABLED --> DISABLED: admin disable
  ENABLED --> EMERGENCY_STOPPED: emergency stop (reason required)
  DISABLED --> EMERGENCY_STOPPED: emergency stop
  EMERGENCY_STOPPED --> DISABLED: clear stop (Admin+, reason)
  EMERGENCY_STOPPED --> ENABLED: clear stop and enable (Admin+, reason, confirmation)
```

Effective playability = `event.status = LIVE` AND `game_settings.state = ENABLED`.

| Effect on | DISABLED | EMERGENCY_STOPPED |
|---|---|---|
| Lobby card | Visible, "temporarily unavailable" (AC-005) | Same |
| New session (`POST /game-sessions`) | `409 GAME_UNAVAILABLE` | `409 GAME_UNAVAILABLE` |
| ISSUED session `start` | `409 GAME_UNAVAILABLE`, session → CANCELLED, **no attempt consumed** | Same |
| STARTED session submit | Accepted normally (preserve data) | Accepted; attempt flagged `DURING_EMERGENCY_STOP` for review |
| Realtime | `game_status_changed` | `game_status_changed` + admin alert |
| Audit | yes | yes, reason mandatory |

Event-wide stop = setting the event to `PAUSED` (same effects for all games).

### Lobby card state (derived, per participant)

| Card state | Condition | Primary action |
|---|---|---|
| `UNAVAILABLE` | not playable | none |
| `IN_PROGRESS` | open STARTED session before deadline | "attempt in progress/interrupted" info |
| `NOT_PLAYED` | playable, `attempts_used = 0` | start |
| `PLAYED_REMAINING` | playable, `0 < attempts_used < allowed` | play again |
| `EXHAUSTED` | `attempts_used ≥ allowed` | none; shows best & rank (completed) |
| `LOADING_ERROR` | client could not load config | retry (never start without server session) |

`allowed = game_settings.attempt_limit + progress.bonus_attempts`. If an admin lowers `attempt_limit` below a participant's `attempts_used`, the card shows `EXHAUSTED`; history is untouched.

## 2. Game session state machine

```mermaid
stateDiagram-v2
  [*] --> ISSUED: POST /game-sessions (checks passed)
  ISSUED --> STARTED: POST /start (checks re-run; attempt created)
  ISSUED --> EXPIRED: now > start_by (no attempt consumed)
  ISSUED --> CANCELLED: game disabled / participant blocked / superseded
  STARTED --> SUBMITTED: result received (accepted OR rejected)
  STARTED --> EXPIRED: now > late_deadline_at without result (attempt ABANDONED)
  SUBMITTED --> [*]
  EXPIRED --> [*]
  CANCELLED --> [*]
```

| Field | Value |
|---|---|
| `id` | UUIDv4 (unguessable, used in URLs) |
| `seed` | 128-bit CSPRNG, hex; returned at ISSUED so assets/layouts can prepare |
| `start_by` | `issued_at + 10 min` (covers slow asset loads) |
| `started_at` | DB time at `/start` |
| `deadline_at` | `started_at + duration_ms + pause_budget_ms + submit_grace_ms` |
| `late_deadline_at` | `deadline_at + late_submission_window_ms` |
| `config_version_id` | Pinned at ISSUED; used for validation even if a new version is published |

Defaults (event policy, OQ-15): `pause_budget_ms = 60 000`, `submit_grace_ms = 20 000`, `late_submission_window_ms = 600 000`.

**Open-session rule (INV-02):** `POST /game-sessions` when an ISSUED session exists for the same participant + game returns that session (idempotent re-entry, e.g., after reload). When a STARTED session exists and is before `late_deadline_at`, it returns `409 SESSION_ALREADY_ACTIVE` with the session id; the participant cannot start a parallel attempt.

## 3. Attempt state machine

An attempt is created at `/start`. It consumes one allowed attempt irrevocably (except by explicit admin bonus).

```mermaid
stateDiagram-v2
  [*] --> IN_PROGRESS: session started (attempt_number assigned)
  IN_PROGRESS --> ACCEPTED: validation passed
  IN_PROGRESS --> ACCEPTED_FLAGGED: passed with soft anomalies
  IN_PROGRESS --> REJECTED: hard validation failure
  IN_PROGRESS --> ABANDONED: no result by late_deadline_at
  ACCEPTED --> INVALIDATED: admin (reason, audited)
  ACCEPTED_FLAGGED --> INVALIDATED: admin (reason, audited)
  ACCEPTED_FLAGGED --> ACCEPTED: admin clears flags (audited)
  INVALIDATED --> ACCEPTED: Super Admin restore (reason, audited)
```

| Status | Counts toward attempts used | Valid for Best Score | Grants base ticket | Sent externally |
|---|---|---|---|---|
| IN_PROGRESS | yes | no | no | no |
| ACCEPTED | yes | yes | if first valid | yes |
| ACCEPTED_FLAGGED | yes | yes | if first valid | yes |
| REJECTED | yes | no | no | no |
| ABANDONED | yes | no | no | no |
| INVALIDATED | yes | no (best recomputed) | ticket handling per OQ-23 | correction delivery per contract (OQ-04) |

On INVALIDATED/restore, the scoring module recomputes `best_score` for that participant + game from remaining valid attempts in the same transaction and emits `best_score_changed` / `leaderboard_changed`.

## 4. Attempt consumption policy (answers SPEC §8.2)

| Situation | Attempt consumed? |
|---|---|
| Server error creating session | No (no session) |
| Asset load fails / participant leaves at pre-game | No (ISSUED expires) |
| `/start` request fails before commit | No (client retries `/start` with same Idempotency-Key) |
| `/start` committed but response lost | Yes, but retry of `/start` returns the same STARTED state — client proceeds normally |
| Page refresh / crash during gameplay | Yes (ABANDONED unless payload was saved after finish) |
| Browser backgrounded during gameplay | Yes, attempt continues under pause budget |
| Network loss during gameplay | Yes; gameplay continues offline; result submitted when network returns (within late window) |
| Game disabled during gameplay | Yes; submission still accepted |
| Result rejected | Yes |

Operator remedy for genuine booth accidents: grant `bonus_attempts` to a participant + game (permission `participant.grant_bonus_attempt`, reason, audit).

## 5. Sequence — game attempt (full pipeline)

```mermaid
sequenceDiagram
  autonumber
  actor P as Participant
  participant C as App (GameHost)
  participant G as Phaser module
  participant API as API
  participant DB as PostgreSQL
  participant RT as Realtime hub
  participant W as Worker
  participant EXT as Snowa API
  P->>C: tap game card (lobby)
  C->>C: import game chunk · show pre-game instructions
  C->>API: POST /game-sessions {game:"spin-perfect"} (Idempotency-Key k1)
  API->>DB: BEGIN · lock progress row · check event LIVE, game ENABLED, attempts_used < allowed, no open STARTED
  API->>DB: INSERT game_session(ISSUED, seed, config_version) · COMMIT
  API-->>C: 201 {sessionId, seed, params, durationMs, startBy}
  C->>G: boot(seed, params) → load critical assets
  G-->>C: ready
  P->>C: press start
  C->>API: POST /game-sessions/:id/start (Idempotency-Key k2)
  API->>DB: BEGIN · lock progress · re-check availability & limit · attempts_used+1 · INSERT attempt(IN_PROGRESS, n) · session STARTED · COMMIT
  API-->>C: 200 {attemptNumber:n, startedAt, deadlineAt}
  C->>C: countdown 3-2-1
  C->>G: start()
  G->>G: gameplay (local, no network)
  G-->>C: finished {claimedScore, actions, activeMs, pausedMs}
  C->>C: save pending-result (localStorage)
  C->>API: POST /game-sessions/:id/result (Idempotency-Key = sessionId)
  API->>DB: BEGIN · lock progress · load session, attempt
  API->>API: validate (binding, timing, replay action log with game-core, bounds, plausibility)
  API->>DB: UPDATE attempt(ACCEPTED, attempt_score) · INSERT attempt_payload, flags
  API->>DB: best = max(best, score) → UPDATE progress (best_score, best_achieved_at, best_attempt_id/seq)
  API->>DB: first valid? INSERT raffle_ticket(BASE) ON CONFLICT DO NOTHING
  API->>DB: SAVEPOINT · evaluate reward rules · atomic inventory · INSERT reward_grants
  API->>DB: INSERT external_delivery(PENDING) · INSERT analytics rows · session SUBMITTED · COMMIT
  API->>DB: SELECT rank (indexed)
  API-->>C: 200 ResultModel {attemptScore, bestScore, isNewBest, rank, baseTicket, extraRewards, attemptsUsed, attemptsAllowed}
  C->>C: clear pending-result · render result
  API->>RT: NOTIFY attempt_accepted, best_score_changed, leaderboard_changed
  RT-->>RT: admin feed / displays / leaderboard viewers
  W->>DB: claim external_delivery (SKIP LOCKED)
  W->>EXT: deliver (adapter)
  EXT-->>W: 2xx
  W->>DB: DELIVERED
```

## 6. Sequence — retry attempt (existing Best Score)

```mermaid
sequenceDiagram
  autonumber
  participant C as App
  participant API as API
  participant DB as PostgreSQL
  Note over DB: progress: attempts_used=1, best_score=6200 (attempt #1), base ticket exists
  C->>API: POST /game-sessions {game}
  API->>DB: attempts_used(1) < allowed(3) → ISSUED
  C->>API: POST /start
  API->>DB: attempts_used=2 · attempt #2 IN_PROGRESS
  C->>API: POST /result (claimed 4100)
  API->>API: replay → 4100 (valid)
  API->>DB: attempt #2 ACCEPTED 4100
  API->>DB: 4100 > 6200? no → best unchanged
  API->>DB: base ticket already exists → no new ticket
  API->>DB: evaluate reward rules (e.g., per-attempt rules may still apply)
  API-->>C: {attemptScore:4100, bestScore:6200, isNewBest:false, baseTicket:{grantedNow:false, active:true}, attemptsUsed:2, attemptsAllowed:3}
  Note over C: Result shows current score separately from best · "entry active" not "ticket granted" (SPEC §13.5)
  C->>API: (later) attempt #3 → 7900
  API->>DB: 7900 > 6200 → best=7900, best_achieved_at=now, best_attempt=#3
  API-->>C: {attemptScore:7900, bestScore:7900, isNewBest:true, rank:…}
```

Equal score to current best (`attempt_score = best_score`) does **not** replace the best; `best_achieved_at` keeps the earlier time (preserves the tie-break advantage of the earlier achievement).
