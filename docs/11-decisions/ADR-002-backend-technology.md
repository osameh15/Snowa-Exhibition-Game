# ADR-002: Backend Technology

Status: **Accepted** (resolves OQ-01)

## Context
Requirements that drive the choice:
- Transactional integrity (attempt limits, best score, idempotent tickets, atomic inventory, immutable draws).
- Server replay of game action logs using **the same deterministic scoring code** as the browser games (ADR-010).
- SSE connections and background outbox processing.
- Small team, short timeline, event-day operability, exhibition-scale load (A-01: ~20 results/s assumed).
- Single inexpensive VPS for v1 ([ADR-012](ADR-012-deployment-topology.md)).

## Options

| Criterion (weight) | Node.js + TS + Fastify (modular monolith) | Nuxt/Nitro server routes | NestJS (Node + TS) | Go (chi/echo) | Laravel (PHP) | ASP.NET Core |
|---|---|---|---|---|---|---|
| Implementation speed (3) | 5 | 5 | 4 | 3 | 4 | 3 |
| Share game-core scoring with client (3) | **5 (same code)** | 5 | 5 | 1 (port + drift risk) | 1 | 1 |
| Real-time SSE support (2) | 5 | 3 | 5 | 5 | 2 | 5 |
| Background jobs (2) | 4 | 2 | 4 | 5 | 4 | 5 |
| Operational simplicity (3) | 4 | 4 (couples UI & API) | 4 | 5 | 3 | 3 |
| Security/maturity (2) | 4 | 3 | 4 | 5 | 4 | 5 |
| Maintainability (2) | 4 | 3 | 5 | 4 | 4 | 4 |
| Team skill fit with TS frontend (2) | 5 | 5 | 4 | 2 | 2 | 2 |
| Weighted total (max 95) | **86** | 74 | 83 | 69 | 56 | 63 |

## Decision
**Node.js 22 LTS + TypeScript + Fastify**, organized as a modular monolith running as **one process** in v1:
- HTTP REST + SSE + an in-process `jobs` module (outbox sender, session sweeper, reward re-evaluation, reconciliation).
- The `jobs` module is a separate backend module with its own entrypoint-ready runner, so it can later run as a dedicated worker process (scaling stage 4) without changing domain logic.
- PostgreSQL via a SQL-first typed layer (Drizzle ORM or Kysely; raw SQL allowed for locking queries), zod schemas from `packages/contracts`, pino logging.
- Supervised by `systemd`; no Docker requirement.

Why not Nitro: the backend is the trust anchor and benefits from an independent lifecycle; the frontend is a static build with no server runtime.
Why not Go/.NET: every game's simulation would need a second implementation — a permanent source of client/server score mismatch.

## Consequences
+ One language end to end; game-core runs identically in Node and browsers (integer math rule).
+ One small process to operate; clean path to separate worker and multiple instances.
− Single event loop: replay is tiny (~1 ms per result); draw selection for very large pools uses a worker thread if > 100 ms; jobs yield between batches so request latency is protected (measured in load tests).
− Module boundaries need lint enforcement.
