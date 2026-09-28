# Analytics and Telemetry

Two separate concerns [SPEC §23]:

| | Business analytics | Operational telemetry |
|---|---|---|
| Purpose | Funnel, engagement, completion, reward/draw reporting | Health, latency, errors, capacity, incident diagnosis |
| Audience | Snowa marketing, event managers (Admin → Reports) | Engineering/DevOps on-call |
| Storage | PostgreSQL (`analytics_events` + transactional tables) | Metrics (Prometheus-compatible), logs (structured JSON), traces (optional) |
| PII | Pseudonymous `participant_id` only; never phone/name/OTP | Never phone/name/OTP; masked phone only where essential |
| Source of truth? | No — transactional tables are | No |
| Third parties | None by default (SPEC §23; OQ-30) | Self-hosted stack |

## 1. Business event taxonomy

Naming: `snake_case`, past tense for facts. Common envelope:

```json
{ "event": "game_completed", "ts_client": "2026-10-01T09:30:12.345Z", "anon_id": "uuid-v4 (device, localStorage)",
  "participant_id": "uuid|null", "session_ref": "game_session_id|null", "src": "booth-a|null",
  "app_version": "1.4.2", "device_class": "phone|tablet|desktop", "props": { } }
```

| Event | Emitted by | Key props | Notes |
|---|---|---|---|
| `qr_entry` | client (landing load with `src`) | `src`, `device_class` | SPEC calls it `qr_landing_view`; single name `qr_entry` used |
| `otp_requested` | **server** | `outcome` (`sent`/`rate_limited`/`send_failed`), `prefix` (e.g., `0912`) | No raw phone/OTP |
| `otp_verified` | **server** | `outcome` (`success`/`wrong_code`/`expired`/`locked`), `is_new_participant` | |
| `profile_completed` | server | `first_time: true` | SPEC `profile_name_saved` |
| `lobby_viewed` | client | `enabled_games`, `ticket_count` | |
| `game_selected` | client | `game` | SPEC `game_start_requested` |
| `game_session_created` | server | `game`, `attempts_remaining` | |
| `game_started` | server | `game`, `attempt_number`, `config_version` | SPEC `game_session_started` |
| `game_completed` | server | `game`, `attempt_score`, `active_ms`, `status` | Emitted when result accepted |
| `attempt_rejected` | server | `game`, `reason_code` | |
| `attempt_abandoned` | server | `game`, `phase` | |
| `best_score_updated` | server | `game`, `old_best`, `new_best`, `rank` | SPEC `best_score_changed` |
| `raffle_ticket_granted` | server | `source` (`BASE`/`REWARD`/`ADMIN`), `game`, `quantity` | |
| `reward_granted` | server | `reward_type`, `reward_definition_id`, `rule_id` | SPEC `extra_reward_granted` |
| `leaderboard_viewed` | client | `game`, `context` (`lobby`/`result`/`nav`) | |
| `raffle_started` | server (admin action) | `draw_id`, `eligible_count`, `winner_count` | Also audit |
| `raffle_completed` | server | `draw_id`, `winner_count` | SPEC `draw_executed` |
| `outbound_api_success` | outbox sender (jobs) | `game`, `attempt_count` | |
| `outbound_api_failure` | outbox sender (jobs) | `category` (`timeout`/`5xx`/`4xx`/`auth`), `final` (bool) | SPEC `external_delivery_status` |
| `pwa_installed` | client | — | optional |

Server-emitted events are written **in the same transaction** as the fact (so counts match), client events via `POST /api/v1/analytics/events` (batched, `sendBeacon` on `pagehide`, max 50 events/batch, rate-limited, unauthenticated allowed with `anon_id`).

## 2. Core metrics (SPEC §23.2) and their source

| Metric | Source |
|---|---|
| Verified participants | `participants` (count where `verified_at` not null) |
| Starts / completions per game, completion rate | `attempts` by status |
| Score avg/median/p90/p99 | `attempts` (valid) |
| Replay rate | participants with `attempts_used > 1` / participants with ≥1 |
| Completed all three | `participant_game_progress` with first valid completion for 3 games |
| Base tickets issued, extra rewards issued, inventory remaining | `raffle_tickets`, `reward_grants`, `reward_rules` |
| Funnel QR → OTP → lobby → first game | `analytics_events` joined by `anon_id`/`participant_id` |
| Draw history | `draws` |
| External delivery success/backlog | `external_deliveries` |

## 3. Operational telemetry

See [Observability](../09-operations/03-observability.md) for metric names, log schema and alerts.
