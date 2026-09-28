# Lobby, Game Configuration, Leaderboard and Rewards API (Participant)

## GET `/api/v1/lobby`

Always read from current server configuration [SPEC §6.4].

```json
{
  "event": { "status": "LIVE", "nameFa": "…" },
  "participant": { "displayName": "علی رضایی" },
  "tickets": { "totalActive": 3, "base": { "spin-perfect": true, "fridge-rush": true, "vision-hunt": false }, "extra": 1 },
  "games": [
    {
      "slug": "spin-perfect", "titleFa": "لحظه طلایی", "sortOrder": 1,
      "cardState": "PLAYED_REMAINING",
      "available": true,
      "attemptsUsed": 1, "attemptsAllowed": 3,
      "bestScore": 8750, "rank": 42,
      "baseTicketEarned": true,
      "openSession": null,
      "assetsVersion": "sp-1.0.0-3f2a"
    }
  ],
  "serverTime": "…"
}
```
- `cardState` per [lobby card states](../02-domain/03-game-session-attempt-lifecycle.md#lobby-card-state-derived-per-participant).
- Games always listed in stable `sortOrder`; disabled games included with `available: false` (AC-005).
- `openSession` = `{sessionId, state}` if ISSUED/STARTED exists.

## GET `/api/v1/games/{slug}`
Static pre-game info: `titleFa`, instruction keys, `durationMs`, asset manifest URL (`/games/spin-perfect/manifest.<hash>.json`), current `configVersion`. Cacheable for 60 s.

## GET `/api/v1/leaderboards/{slug}?limit=50&cursor=…`

```json
{
  "game": "spin-perfect",
  "asOf": "…",
  "items": [ { "rank": 1, "publicName": "…", "score": 12540, "isMe": false } ],
  "me": { "rank": 42, "score": 8750, "publicName": "…" },
  "nextCursor": "opaque"
}
```
- `limit` ≤ 100; maximum depth 1000 for participants.
- Public projection only (no ids, no phone).
- Served from 1 s per-process top-N cache for the first page.

## GET `/api/v1/leaderboards/{slug}/stream` (SSE)
Emits `leaderboard_changed` hints; see [Real-time events](07-realtime-events.md). Client closes it when hidden.

## GET `/api/v1/me/rewards`
```json
{ "items": [ { "grantId": "uuid", "type": "DISCOUNT_CODE", "titleFa": "…", "code": "…", "status": "GRANTED",
               "sourceGame": "fridge-rush", "grantedAt": "…", "validUntil": "…" } ] }
```

## GET `/api/v1/me/tickets`
```json
{ "totalActive": 4, "items": [ { "source": "BASE", "game": "spin-perfect", "grantedAt": "…" }, { "source": "REWARD", "rewardTitleFa": "…", "grantedAt": "…" } ] }
```

## POST `/api/v1/analytics/events`
Batch ≤ 50 events (see [Analytics](../01-architecture/13-analytics-and-telemetry.md)). Anonymous allowed. `202`. Rate limited per `anon_id` and IP class. Unknown event names rejected.
