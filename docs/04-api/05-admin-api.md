# Admin API

Base: `/api/admin/v1`. Auth: admin session cookie with completed MFA. Every mutating endpoint: `Idempotency-Key` required, entity `version` required where the entity is versioned, `reason` required where marked ⚠, audit row written. Permissions: [Roles & permissions](../06-admin/02-roles-and-permissions.md).

## 1. Authentication

| Method & path | Permission | Notes |
|---|---|---|
| `POST /auth/login` `{username, password}` | — | → `{mfaRequired:true, mfaToken}` (short-lived); generic error on bad credentials; lockout after 5 failures / 15 min |
| `POST /auth/mfa` `{mfaToken, totp}` | — | Sets `sx_as` cookie |
| `POST /auth/logout` | any | |
| `GET /auth/me` | any | role, permissions list (UI gating only) |

## 2. Dashboard and live

| Endpoint | Permission | Returns |
|---|---|---|
| `GET /dashboard` | `dashboard.view` | KPIs (online, playing now, verified participants, accepted attempts, avg play time, tickets issued, rewards issued), per-game distribution, health (outbox backlog, OTP success rate 5 min, SSE clients), `lastEventId`, `asOf` |
| `GET /stream` (SSE) | `dashboard.view` | admin channel events |
| `GET /activity?limit=&cursor=` | `dashboard.view` | recent accepted attempts/new bests/grants/draws (masked PII) |
| `GET /live-scores?game=&limit=` | `attempt.view` | latest accepted attempts |

## 3. Games

| Endpoint | Permission | Body / notes |
|---|---|---|
| `GET /games` | `game.view` | settings + metrics (players, attempts, avg/best score) + active config version |
| `PATCH /games/{slug}/state` | `game.toggle` | `{state: ENABLED\|DISABLED, version}` |
| `POST /games/{slug}/emergency-stop` ⚠ | `game.emergency_stop` | `{reason}` |
| `POST /games/{slug}/emergency-clear` ⚠ | `game.emergency_stop` (Admin+) | `{targetState, reason, version}` |
| `PATCH /games/{slug}/attempt-limit` | `game.set_attempt_limit` | `{attemptLimit: 1..100, version}`; response includes impact preview counts (participants who become exhausted / regain attempts) |
| `GET /games/{slug}/config-versions` | `game.view` | list |
| `POST /games/{slug}/config-versions/{id}/publish` ⚠ | `game.publish_config` (Super Admin) | Blocked while event LIVE unless `force` + reason (scores become non-comparable) |
| `POST /event/status` ⚠ | `event.manage` | `{status: LIVE\|PAUSED\|CLOSED, reason}` |

## 4. Rewards

| Endpoint | Permission |
|---|---|
| `GET/POST /reward-definitions`, `PATCH /reward-definitions/{id}` | `reward.view` / `reward.manage` |
| `GET/POST /reward-rules`, `PATCH /reward-rules/{id}` | `reward.manage` (server rejects unsafe rules: `422 RULE_UNSAFE`; unlimited requires `reward.create_unlimited`) |
| `POST /reward-rules/{id}/activate` / `pause` | `reward.manage` |
| `POST /reward-definitions/{id}/codes` (CSV upload, ≤ 50k lines) | `reward.upload_codes` → `{inserted, duplicates, invalid}` |
| `GET /reward-grants?status=&definition=&cursor=` | `reward.view` |
| `PATCH /reward-grants/{id}` `{status, note, version}` | `reward.fulfill` (fulfillment transitions) |
| `POST /reward-grants/{id}/revoke` ⚠ | `reward.manage` `{reason, returnToInventory}` |

## 5. Participants, attempts, tickets

| Endpoint | Permission |
|---|---|
| `GET /participants?q=&cursor=` (name trigram / phone suffix/exact) | `participant.view` (phones masked) |
| `GET /participants/{id}` | `participant.view` — profile, progress per game, attempts, tickets, grants, deliveries, audit excerpts |
| `POST /participants/{id}/reveal-phone` | `participant.view_phone` (audited each time) |
| `POST /participants/{id}/block` / `unblock` ⚠ | `participant.block` |
| `POST /participants/{id}/hide-name` / `PATCH …/display-name` ⚠ | `participant.moderate_name` |
| `POST /participants/{id}/games/{slug}/bonus-attempts` ⚠ `{count: 1..3}` | `participant.grant_bonus_attempt` |
| `GET /attempts?game=&status=&flag=&cursor=` | `attempt.view` |
| `GET /attempts/{id}` | `attempt.view` — incl. payload summary, recomputed stats, flags, trace |
| `POST /attempts/{id}/invalidate` ⚠ | `attempt.invalidate` |
| `POST /attempts/{id}/clear-flags` ⚠ | `attempt.invalidate` |
| `POST /attempts/{id}/restore` ⚠ | `attempt.restore` (Super Admin) |
| `POST /tickets/{id}/void` / `reinstate` ⚠ | `ticket.void` / Super Admin |
| `POST /participants/{id}/tickets` ⚠ `{quantity}` | `ticket.grant` (Super Admin) |
| `POST /participants/{id}/games/{slug}/exclude-ranking` ⚠ | `leaderboard.exclude` |

Manual score editing is **not** provided. Corrections happen by invalidating/restoring attempts (keeps history honest) — OQ-31 confirms.

## 6. Leaderboards, displays, reports, audit, integration

| Endpoint | Permission |
|---|---|
| `GET /leaderboards/{slug}?cursor=` | `leaderboard.view` (admin projection) |
| `GET/POST /displays`, `POST /displays/{id}/rotate-token`, `DELETE /displays/{id}` | `display.manage` |
| `PATCH /displays/{id}/mode` `{mode: leaderboard\|rotate\|draw\|idle, game?, drawId?}` | `display.manage` / `draw.present` |
| `GET /reports/{name}?from=&to=&game=` | `report.view` |
| `POST /exports` `{report, filters, includePhone}` | `data.export` (`includePhone` requires `participant.view_phone`) → async job; `GET /exports/{id}` → one-time download |
| `GET /audit?actor=&action=&target=&cursor=` | `audit.view_own` / `audit.view_all` |
| `GET /integrations/deliveries?status=&cursor=` | `integration.view` |
| `GET /integrations/summary` | `integration.view` — counts by status, oldest pending age, last success, error categories |
| `POST /integrations/deliveries/{id}/retry` | `integration.retry` |
| `POST /integrations/deliveries/retry-failed` ⚠ | `integration.bulk_retry` |
| `POST /integrations/deliveries/{id}/resolve` ⚠ | `integration.retry` |
| `PATCH /integrations/snowa` ⚠ (enable/disable delivery, adapter config non-secret parts) | `integration.configure` |
| `GET/POST/PATCH /admin-users` | `admin.manage` |

Raffle endpoints: [Raffle API](06-raffle-api.md).
