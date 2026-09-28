# Real-Time Event Contracts

Transport: SSE ([ADR-003](../11-decisions/ADR-003-realtime-transport.md)). Events are **hints**; each lists the REST resource to re-fetch.

## 1. Wire format

```text
id: 1727512212345-17
event: leaderboard_changed
data: {"v":1,"ts":"2026-10-01T09:30:12.345Z","game":"spin-perfect","topChanged":true}

: ping
```

- `id` = `<epochMs>-<replicaSeq>`; clients send `Last-Event-ID` on reconnect.
- `retry: 3000` sent on connect.
- `v` = payload schema version.
- Server sends `event: resync` when it cannot replay from the given id → client re-fetches all snapshots for that view.
- Server sends `event: hello` with `{channel, serverTime, buildHash}` on connect (displays use `buildHash` to schedule safe reloads).

## 2. Channels

| Channel | Endpoint | Auth | Events |
|---|---|---|---|
| admin | `/api/admin/v1/stream` | admin | all admin events below |
| display | `/api/display/v1/stream` | display | `display_mode_changed`, `leaderboard_changed`, `draw_presentation_started`, `draw_winner_revealed`, `draw_voided`, `system_status_changed` |
| leaderboard | `/api/v1/leaderboards/{slug}/stream` | participant | `leaderboard_changed`, `game_status_changed` |

## 3. Event catalog

| Event | Payload (`data`) | Channels | Client action |
|---|---|---|---|
| `participant_joined` | `{count}` (aggregate only) | admin | update KPI |
| `attempt_accepted` | `{game, attemptId, publicName, score, isNewBest, at}` | admin | prepend to live feed; no re-fetch |
| `attempt_rejected` | `{game, attemptId, reason}` | admin | feed (ops visibility) |
| `best_score_changed` | `{game, participantId, bestScore}` | admin | refresh participant detail if open |
| `leaderboard_changed` | `{game, topChanged}` (coalesced ≤ 1/s/game) | all | re-fetch `GET …/leaderboards/{game}` if `topChanged` or own view affected |
| `game_status_changed` | `{game, state}` | admin, leaderboard | re-fetch games / lobby |
| `game_attempt_limit_changed` | `{game, attemptLimit}` | admin | re-fetch games |
| `reward_definition_changed` | `{definitionId}` / `reward_rule_changed {ruleId, status}` | admin | re-fetch rewards |
| `reward_inventory_low` | `{ruleId, remaining}` (≤ 10% or ≤ 10 units) | admin | alert |
| `draw_started` | `{drawId}` (execution requested) | admin | lock UI |
| `draw_completed` | `{drawId, winnerCount}` | admin | re-fetch draw |
| `draw_presentation_started` | `{drawId, winnerCount, eligibleCount}` | display, admin | start animation |
| `draw_winner_revealed` | `{drawId, position, revealedCount}` | display, admin | `GET /display/v1/draws/{id}` then land animation |
| `draw_voided` | `{drawId}` | display, admin | exit draw mode |
| `display_mode_changed` | `{displayId?, mode, game?, drawId?}` | display | fetch `/state` |
| `metrics_tick` | `{online, playingNow, attemptsLastMin, participantsTotal}` every 10 s | admin | KPI update |
| `delivery_backlog` | `{pending, failed, oldestPendingAgeS}` every 30 s or on threshold change | admin | integration badge |
| `system_status_changed` | `{component: otp\|outbox\|db\|realtime, status: ok\|degraded\|down}` | admin, display (subset) | banner |

No event contains phone numbers or OTP data. `publicName` follows the public identity policy.

## 4. Client recovery algorithm

```text
on connect/reconnect:
    fetch snapshot(s) for the current view (REST) → store asOf
    apply subsequent events only if event.ts > asOf (ignore older)
on 'resync' or gap suspected (e.g., > 35 s silence):
    show 'reconnecting'; re-fetch snapshots
after 3 consecutive failed reconnects:
    fallback: poll snapshot every 10 s; retry SSE every 60 s
```
