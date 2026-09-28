# ADR-003: Real-Time Transport

Status: **Accepted** (lightweight model confirmed in the final Phase 0 correction)

## Context
Near-real-time updates are needed for admin live scores, leaderboards, participant counts, public displays and raffle presentation [SPEC §19]. Messages are hints; the DB is authoritative. Traffic is one-directional (server → client); commands use REST.

## Options
| Option | Pros | Cons |
|---|---|---|
| Polling only | Simplest | Latency vs load trade-off; draws feel laggy |
| **SSE** | One-way fits exactly; plain HTTP; cookies; built-in reconnect + `Last-Event-ID` | Proxy buffering must be off |
| WebSocket | Bidirectional | Extra infrastructure and logic; bidirectionality unused |
| Managed realtime service | Offloads scale | External dependency, likely unreachable in target hosting (A-05) |

## Decision
REST snapshots + SSE + polling fallback. SSE runs **in the same Fastify process** in v1; domain modules publish through a `RealtimeBus` port implemented in-process. For multiple Fastify instances (scaling stage 5) the bus adapter switches to PostgreSQL `LISTEN/NOTIFY`; Redis pub/sub only if measurements require (stage 6). No Redis or WebSocket infrastructure in v1.

## Consequences
+ Minimal code and zero extra infrastructure.
+ Draw reveal is a persisted REST command + SSE hint → resilient to disconnects and restarts.
− A process restart drops streams briefly; clients reconnect and `resync`.
