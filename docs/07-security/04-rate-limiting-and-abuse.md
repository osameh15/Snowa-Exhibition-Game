# Rate Limiting and Abuse Controls

Two layers: coarse limits at the reverse proxy (protect capacity) and semantic limits in the API (protect business rules). v1 stores API counters in PostgreSQL (`rate_limit_buckets` with upsert, or derived from domain tables); Redis is the upgrade path if load tests show contention ([ADR-004](../11-decisions/ADR-004-persistence.md)).

| Scope | Endpoint(s) | Limit (default, tunable live via event policy) | Response |
|---|---|---|---|
| Proxy: per IP | all `/api` | 50 req/s burst 100 (NAT-friendly) | 429 |
| Proxy: connections | SSE | 20 concurrent per IP (displays exempt by allowlist) | 429 |
| Proxy: body size | all | 128 KB max | 413 |
| OTP request | per phone | 5 / 15 min, 10 / day | 429 `retryAfterMs` |
| OTP request | per IP class | 60 / 10 min | 429 |
| OTP verify | per challenge | 5 attempts | 422 `OTP_LOCKED` |
| OTP verify | per phone | 20 fails / day → 1 h block | 429 |
| Game session create | per participant | 10 / min | 429 |
| Result submit | per participant | 10 / min | 429 |
| Leaderboard fetch | per session | 60 / min | 429 |
| Analytics ingest | per anon_id | 20 batches / min | 202 (dropped silently beyond) |
| Admin login | per username & per IP | 5 / 15 min | 429 + lockout |
| Draw preview | per admin | 1 / s | 429 |
| Export | per admin | 5 / hour | 429 |

Abuse responses are Persian cooldown screens with countdowns (never raw errors). All limit hits are counted in metrics (`rate_limited_total{scope}`) and visible on the ops dashboard to tune limits live if venue NAT causes false positives.
