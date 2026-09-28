# ADR-002: Backend Technology

Status: **Proposed** — backend technology is not yet approved (OQ-01). No existing repository code constrains the choice (the repository contains only `docs/source`).

## Context
Requirements that drive the choice:
- Transactional integrity (attempt limits, best score, idempotent tickets, atomic inventory, immutable draws).
- Server replay of game action logs using **the same deterministic scoring code** as the browser games (ADR-010).
- Long-lived SSE connections, background outbox worker.
- Small team, short timeline, event-day operability, exhibition-scale load (A-01: ~20–100 results/s).
- Hosting unknown (OQ-02); possibly domestic Iranian infrastructure with limited managed services.

## Options

| Criterion (weight) | Node.js + TS + Fastify (modular monolith) | Nuxt/Nitro server routes (same app) | NestJS (Node + TS) | Go (chi/echo) | Laravel (PHP) | ASP.NET Core |
|---|---|---|---|---|---|---|
| Implementation speed (3) | 5 | 5 | 4 | 3 | 4 | 3 |
| Share game-core scoring with client (3) | **5 (same code)** | 5 | 5 | 1 (port + drift risk) | 1 | 1 (or run JS engine) |
| Real-time SSE support (2) | 5 | 3 (tied to Nitro runtime/presets) | 5 | 5 | 2 (PHP-FPM ill-suited to long connections) | 5 |
| Background workers (2) | 4 (separate entrypoint) | 2 | 4 | 5 | 4 (queues) | 5 |
| Operational simplicity (3) | 4 | 4 (1 deployable) but couples UI & API deploys | 4 | 5 (single binary) | 3 | 3 |
| Security/maturity (2) | 4 | 3 | 4 | 5 | 4 | 5 |
| Maintainability (2) | 4 (with module boundaries) | 3 | 5 (opinionated) | 4 | 4 | 4 |
| Team skill fit with TS frontend (2) | 5 | 5 | 4 | 2 | 2 | 2 |
| Weighted total (max 95) | **86** | 74 | 83 | 69 | 56 | 63 |

(Scores 1–5 × weight; indicative, to be validated against the actual team.)

## Decision (recommended)
**Node.js 22 LTS + TypeScript + Fastify**, organized as a modular monolith with two entrypoints (`api`, `worker`), PostgreSQL via a SQL-first typed layer (Drizzle ORM or Kysely; raw SQL allowed for locking queries), zod schemas shared from `packages/contracts`, pino logging.

Why not Nitro: it would couple API deployment and scaling with the frontend build and make the worker/SSE story depend on Nitro presets; the backend is the trust anchor and benefits from an independent lifecycle.
Why not Go/.NET: excellent runtimes, but scoring would need a second implementation of every game's simulation — a permanent source of client/server mismatch (false `SCORE_MISMATCH` rejections).
NestJS is an acceptable alternative if the team prefers stronger conventions; the documented behavior does not change.

## Consequences
+ One language end to end; game-core runs identically in Node and browsers (integer math rule).
+ Fast delivery; mature ecosystem for SSE, Postgres, testing.
− Single-threaded event loop: CPU-heavy replay must stay small (action logs are tiny; replay ~1 ms). Draw selection for very large pools runs in a worker thread if > 100 ms.
− Requires discipline for module boundaries (lint rules on imports).
