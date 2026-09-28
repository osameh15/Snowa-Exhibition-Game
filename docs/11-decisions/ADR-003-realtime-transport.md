# ADR-003: Real-Time Transport

Status: **Proposed**

## Context
Near-real-time updates are needed for admin live scores, leaderboards, participant counts, public displays and raffle presentation [SPEC §19]. Messages are hints; the DB is authoritative. Traffic is one-directional (server → client); commands already use REST.

## Options
| Option | Pros | Cons |
|---|---|---|
| Polling only | Simplest, cache-friendly | Latency vs load trade-off; many requests from displays/admin; draws feel laggy |
| **SSE** | One-way fits exactly; HTTP semantics (cookies, proxies, HTTP/2); built-in reconnect + `Last-Event-ID`; trivial server code | No client→server channel (not needed); needs proxy buffering off; HTTP/1.1 6-connection limit (mitigated by HTTP/2) |
| WebSocket (raw / Socket.IO) | Bidirectional, widespread | More moving parts (upgrade handling, heartbeat, reconnection/resume logic, sticky sessions for Socket.IO), bidirectionality unused |
| Managed realtime service | Offloads scale | External dependency, likely unavailable/unreliable in target hosting (A-05), cost |

## Decision
SSE for admin, display and (while visible) participant leaderboard; REST snapshot endpoints for all views; polling fallback after repeated SSE failure. Participants do not hold persistent connections elsewhere (battery, background tab killing, venue network churn). Cross-replica fan-out via PostgreSQL `LISTEN/NOTIFY`; Redis pub/sub only if replicas or volume outgrow it.

## Consequences
+ Minimal code and infrastructure; robust reconnect; works with standard proxies.
+ Draw reveal is a REST command + SSE hint, persisted state → resilient.
− Proxy must be configured for streaming (buffering off, long read timeouts).
− If a future feature needs client→server streaming (not foreseen), revisit.
