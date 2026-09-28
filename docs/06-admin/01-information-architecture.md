# Admin Information Architecture

Source: SPEC §16, concept CA-09. Architecture: [Admin architecture](../01-architecture/06-admin-architecture.md). Language: Persian/RTL.

## 1. Navigation tree

```text
داشبورد (Dashboard)                     live KPIs, game strip, live leaderboard, activity, health, quick draw
مدیریت بازی‌ها (Games)
  ├─ Game detail (per game)             state, attempt limit, core reward (read-only), extra rewards, metrics, config version
  └─ Emergency controls                 stop game / pause event
قرعه‌کشی (Raffle)
  ├─ Draws list (history)
  ├─ New draw (filter builder → preview → execute)
  └─ Presentation console               present / reveal next / reveal all
شرکت‌کنندگان (Participants)
  ├─ Search/list
  └─ Participant detail                 profile, per-game progress, attempts, tickets, rewards, deliveries, audit
تلاش‌ها (Attempts)                       live scores, flagged review queue, attempt detail
جدول رتبه‌بندی (Leaderboards)            per-game full leaderboard, exclusions
جوایز و کدهای تخفیف (Rewards & Codes)
  ├─ Definitions
  ├─ Rules
  ├─ Code pools (upload, stock)
  └─ Grants & fulfillment
نمایشگرها (Displays)                     devices, tokens, modes
گزارش‌ها (Reports)                       predefined reports, exports
یکپارچه‌سازی (Integrations)              Snowa delivery monitor, OTP provider health
گزارش ممیزی (Audit)
تنظیمات رویداد (Event settings)          event status, policies (Super Admin), admin users
```

## 2. Screen → capability → API mapping

| Screen | SPEC | Primary APIs | Real-time |
|---|---|---|---|
| Dashboard | §16.1 | `GET /dashboard`, `/activity` | admin stream: `metrics_tick`, `attempt_accepted`, `leaderboard_changed`, `delivery_backlog`, `system_status_changed` |
| Games | §16.2 | `GET /games`, `PATCH …/state`, `…/attempt-limit`, `…/emergency-stop` | `game_status_changed`, `game_attempt_limit_changed` |
| Raffle | §15, §16.5 | [Raffle API](../04-api/06-raffle-api.md) | `draw_*` |
| Participants | §16.3 | `/participants*` | — (refresh on open) |
| Attempts / live scores | §14.3 | `/attempts`, `/live-scores` | `attempt_accepted`, `attempt_rejected` |
| Leaderboards | §14 | `/leaderboards/{slug}` | `leaderboard_changed` |
| Rewards | §16.4 | `/reward-*` | `reward_*`, `reward_inventory_low` |
| Displays | §14.2 | `/displays*` | — |
| Reports | §23.2 | `/reports`, `/exports` | — |
| Integrations | §17.2 | `/integrations/*` | `delivery_backlog` |
| Audit | §16, §21 | `/audit` | — |

## 3. Dashboard KPI definitions

| KPI | Definition |
|---|---|
| Online participants | distinct participants with an authenticated request in the last 2 min (AMB-10) |
| Playing now | sessions `STARTED` with `now < deadline_at` |
| Verified participants | participants with `verified_at` not null (event scope) |
| Accepted attempts | attempts `ACCEPTED`/`ACCEPTED_FLAGGED` |
| Average play time | mean `active_ms` of accepted attempts (per game and overall) |
| Tickets issued | active tickets (base / extra split) |
| Participation by game | distinct participants with ≥ 1 valid attempt per game |
| Completed all three | participants with first valid completion in all games |
| Outbox backlog | deliveries PENDING + RETRY_SCHEDULED; oldest age |
| OTP health | send success rate, last 5 min |
