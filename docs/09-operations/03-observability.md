# Observability: Logs, Metrics, Alerting

Stack recommendation: self-hosted Prometheus-compatible metrics + Grafana, Loki (or equivalent) for logs, optional OpenTelemetry traces; error tracking self-hosted (e.g., GlitchTip/Sentry self-hosted) — all self-hosted per A-05. Final choice with hosting (OQ-02).

## 1. Structured logs

JSON lines (pino). Fields: `ts`, `level`, `service` (api/worker), `requestId`, `route`, `status`, `durationMs`, `actorType`, `participantId` (pseudonymous), `adminId`, `sessionId`, `event`, `errorCode`.

Redaction (enforced by logger config + tests):
| Never log | Mask |
|---|---|
| OTP codes, session tokens, cookies, Authorization headers, SMS/Snowa credentials, request bodies of auth/result endpoints, display tokens | phone → `+98912•••6789`; IP → prefix |

Retention: 14–30 days (OQ-13).

## 2. Metrics

| Metric | Type | Labels |
|---|---|---|
| `http_requests_total`, `http_request_duration_seconds` | counter/histogram | route, method, status |
| `otp_requests_total` | counter | outcome |
| `otp_send_duration_seconds` | histogram | provider |
| `otp_verify_total` | counter | outcome |
| `game_sessions_total` | counter | game, event (`issued/started/submitted/expired/cancelled`) |
| `attempts_total` | counter | game, status |
| `attempt_reject_total` / `attempt_flag_total` | counter | game, code |
| `result_pipeline_duration_seconds` | histogram | game |
| `best_score_updates_total` | counter | game |
| `tickets_granted_total` | counter | source |
| `reward_grants_total`, `reward_inventory_remaining` | counter/gauge | rule |
| `outbox_pending`, `outbox_failed`, `outbox_oldest_pending_seconds` | gauge | kind |
| `outbox_deliveries_total` | counter | result (success/retry/permanent) |
| `sse_connections` | gauge | channel |
| `sse_events_sent_total` | counter | type |
| `db_pool_in_use`, `db_query_duration_seconds`, `db_tx_retries_total` | gauge/histogram/counter | — |
| `rate_limited_total` | counter | scope |
| `draws_total` | counter | state |
| Client RUM (sampled, via analytics endpoint) | — | device class: LCP, INP, game FPS p50/p5, asset load time |

## 3. Alerts

| Alert | Condition | Severity |
|---|---|---|
| API down | health check fails 1 min | Sev-1 |
| Error rate | 5xx > 2 % for 5 min | Sev-1 |
| Result latency | p95 result pipeline > 1 s for 5 min | Sev-2 |
| DB | pool saturation > 90 % 2 min; replication lag > 30 s; disk > 80 % | Sev-2 |
| OTP | send success < 90 % over 5 min, or p95 send > 10 s | Sev-1 during event hours |
| Outbox | oldest pending > 15 min or failed > 0 new in 10 min | Sev-3 (Sev-2 if > 1 h) |
| Rejections spike | `SCORE_MISMATCH` > 1 % of results 10 min | Sev-2 (client bug or cheat) |
| Flag rate | flagged > 5 % per game 15 min | Sev-3 |
| Reward inventory | ≤ 10 % remaining | info → operators |
| Admin security | ≥ 5 failed admin logins 5 min; T4 action outside event hours; bulk export | Sev-2 |
| Audit chain | verification failure | Sev-1 |
| SSE | connections drop > 50 % in 1 min | Sev-2 |
| Certificates | expiry < 14 days | Sev-3 |

Routing: event-day on-call phone + ops chat channel (tooling TBD).

## 4. Event-day dashboard (Grafana)

Rows: traffic & errors; OTP funnel; games (sessions/attempts/rejections/flags per game); result latency; DB health; outbox; SSE; rewards inventory; client RUM.
