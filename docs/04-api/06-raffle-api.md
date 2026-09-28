# Raffle API

Domain: [Live raffle](../02-domain/07-live-raffle.md).

## 1. Admin endpoints (`/api/admin/v1`)

| Method & path | Permission | Body | Result |
|---|---|---|---|
| `GET /draws?state=&cursor=` | `draw.view` | — | history incl. voided |
| `POST /draws` | `draw.manage` | `{nameFa, criteria, weighting, winnerCount, allowFewer}` | DRAFT |
| `PATCH /draws/{id}` | `draw.manage` | same fields + `version` | DRAFT only |
| `POST /draws/{id}/preview` | `draw.manage` | — | `{eligibleCount, totalWeight, sample: [publicName…≤10], asOf}` (rate limit 1/s per admin) |
| `POST /draws/{id}/execute` ⚠ | `draw.execute` | `{version, confirmation: {winnerCount, eventSlug}}` + `Idempotency-Key` | COMPLETED with winners (admin projection) |
| `POST /draws/{id}/cancel` | `draw.manage` | `{reason}` | CANCELLED |
| `POST /draws/{id}/present` | `draw.present` | `{displayIds?: []}` | PRESENTING; sets displays to draw mode |
| `POST /draws/{id}/reveal-next` | `draw.present` | `{expectedRevealedCount}` (guards double-click: must equal current) | `{revealedCount}` |
| `POST /draws/{id}/reveal-all` | `draw.present` | — | REVEALED |
| `POST /draws/{id}/void` ⚠ | `draw.void` (Super Admin) | `{reason}` | VOIDED |
| `GET /draws/{id}` | `draw.view` | — | full record; winners with masked phone; full phone via reveal-phone permission |
| `GET /draws/{id}/entries.csv` | `data.export` | — | pseudonymized snapshot for audit |
| `GET /draws/{id}/verification` | `draw.view` | — | `{snapshotHash, seedHex (after COMPLETED), algorithmVersion, recomputedMatches: true}` |

### Criteria schema (JSON)
```json
{
  "population": { "type": "COMPLETED_ALL" },
  "score": { "game": "spin-perfect", "op": "GTE", "value": 5000 },
  "rankRange": { "game": "spin-perfect", "from": 1, "to": 1000 },
  "tickets": { "minCount": 1 },
  "excludePreviousWinners": true
}
```
All keys optional except `population`; present keys are AND-combined. Unknown keys rejected.

### Execute response
```json
{
  "drawId": "uuid", "state": "COMPLETED", "executedAt": "…", "executedBy": "admin-username",
  "eligibleCount": 1843, "totalWeight": 4977, "previewCount": 1839,
  "winners": [ { "position": 1, "participantId": "uuid", "displayName": "…", "phoneMasked": "0912•••6789", "tickets": 4 } ],
  "snapshotHash": "hex", "algorithmVersion": "weighted-wor-hmac-sha256-v1"
}
```

## 2. Display endpoints (`/api/display/v1`)

| Endpoint | Returns |
|---|---|
| `POST /auth` `{token}` | sets `sx_ds` cookie |
| `GET /state` | `{mode, game?, drawId?, asOf}` |
| `GET /leaderboards/{slug}?limit=20` | public projection |
| `GET /draws/{id}` | `{state, winnerCount, eligibleCount, revealedCount, revealed: [{position, publicName}]}` — **only positions ≤ revealedCount** |
| `GET /stream` (SSE) | display channel |
