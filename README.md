# Snowa Exhibition Game

A mobile-first, Persian RTL exhibition gaming platform for Snowa: three score-based branded games, live leaderboards, configurable rewards, raffle tickets, and server-side live raffle management.

## Project Status

| Phase | Status |
|---|---|
| Phase 0 — Architecture & Technical Documentation | **Complete** |
| Phase 1 — Implementation | **Not Started** |

This repository currently contains documentation only. No application code exists yet.

## Product Overview

- **QR-first exhibition experience** — visitors scan a booth QR code and play directly in the mobile browser; PWA installation is optional, never required.
- **Phone + OTP authentication**, followed by **full-name collection on first visit only**.
- **Persian RTL participant UI**, mobile-first, fully supported on tablets.
- **Three initial games**, each built around a Snowa product:
  - **Spin Perfect** — washing machine, timing/precision
  - **Fridge Rush** — refrigerator, rapid sorting/placement
  - **Vision Hunt** — television, reaction/visual search
- **Best-score leaderboards** per game (official score = best valid attempt; earlier achiever wins ties).
- **Raffle tickets** — one base ticket per game on first valid completion.
- **Configurable rewards** — discount codes, extra tickets, physical prizes and other benefits, managed without redeployment.
- **Live admin control room** — game availability, attempt limits, rewards, participants, live scores, audit.
- **Live raffle** — filterable eligibility, server-side winner selection, public display presentation.
- **External Snowa API integration** — accepted results delivered server-to-server with reliable retries.

## Architecture Direction

Status labels follow the [ADR index](docs/11-decisions/README.md): the frontend baseline is **Accepted**; the other decisions below are **Proposed** and awaiting final approval.

**Frontend** (Accepted baseline — [ADR-001](docs/11-decisions/ADR-001-frontend-architecture.md), [ADR-006](docs/11-decisions/ADR-006-game-runtime.md))
- Nuxt 4, Vue 3, TypeScript
- Phaser 3 for gameplay only; all other screens are normal web UI
- Lazy-loaded game modules and per-game assets

**Backend** (Proposed — [ADR-002](docs/11-decisions/ADR-002-backend-technology.md))
- Node.js 22, TypeScript, Fastify
- Modular service with an API process and a worker process

**Data** (Proposed — [ADR-004](docs/11-decisions/ADR-004-persistence.md))
- PostgreSQL as the single stateful dependency
- Redis intentionally deferred unless load testing proves it necessary

**Realtime** (Proposed — [ADR-003](docs/11-decisions/ADR-003-realtime-transport.md))
- Server-Sent Events for live updates
- REST snapshot endpoints for state recovery
- Polling fallback where streaming is unavailable

**Integrity** (Proposed — [ADR-007](docs/11-decisions/ADR-007-reward-architecture.md), [ADR-008](docs/11-decisions/ADR-008-external-integration.md), [ADR-009](docs/11-decisions/ADR-009-authentication-sessions.md), [ADR-010](docs/11-decisions/ADR-010-score-validation.md), [ADR-011](docs/11-decisions/ADR-011-raffle-selection.md))
- Server-issued game sessions
- Server-side score replay and validation
- Best score, tickets and rewards decided by the backend
- Auditable, reproducible raffle execution
- Durable outbox for external API delivery

## Repository Structure

```text
docs/
  00-overview/      technical overview, scope, glossary, traceability matrix
  01-architecture/  system context, components, client, game runtime, backend, realtime, security boundaries
  02-domain/        domain model, lifecycles, best score, tickets, rewards, live raffle, audit
  03-data/          schema, entity definitions, invariants, retention
  04-api/           internal APIs, realtime events, external Snowa adapter contract
  05-games/         shared runtime and per-game technical specifications
  06-admin/         admin information architecture, roles, live operations
  07-security/      threat model, authentication, anti-cheat, privacy
  08-quality/       testing strategy, device matrix, acceptance
  09-operations/    environments, deployment, observability, event-day runbook
  10-performance/   budgets, caching, realtime performance
  11-decisions/     architecture decision records
  12-planning/      Phase 1 plan, milestones, risks, open questions
  images/           concept art boards
  source/           original approved product/game specification
```

`docs/source/` contains the original approved Product & Game Design Specification. It is the product source of truth and **must not be modified**.

## Documentation

Phase 0 contains the implementation-ready architecture. Start from the documentation index: **[docs/README.md](docs/README.md)**.

Key starting documents:
- [Technical overview](docs/00-overview/01-technical-overview.md)
- [Containers & components (architecture)](docs/01-architecture/02-containers-and-components.md)
- [ADR index](docs/11-decisions/README.md)
- [Open questions & assumptions](docs/12-planning/04-open-questions.md)
- [Phase 1 implementation plan](docs/12-planning/01-phase-1-implementation-plan.md)
- [Definition of Ready](docs/12-planning/05-definition-of-ready.md)

## Language

- Technical documentation: English
- Participant-facing application: Persian
- Layout: RTL
- Technical identifiers and source code: English

## Development

Application scaffolding will be introduced in Phase 1. See the [Phase 1 implementation plan](docs/12-planning/01-phase-1-implementation-plan.md) for the approved sequence.

## Phase 1 Gate

Phase 1 starts when these blocking items are resolved (see [Definition of Ready](docs/12-planning/05-definition-of-ready.md#2-ready-to-start-phase-1-when)):

- **OQ-01** — backend technology approval ([ADR-002](docs/11-decisions/ADR-002-backend-technology.md))
- Approval of [ADR-004](docs/11-decisions/ADR-004-persistence.md) (database), [ADR-009](docs/11-decisions/ADR-009-authentication-sessions.md) (sessions) and [ADR-010](docs/11-decisions/ADR-010-score-validation.md) (score replay)
- **OQ-02** — hosting provider, region and managed PostgreSQL availability
- **OQ-15** — product approval of the attempt consumption and interruption policy

## License / Confidentiality

Proprietary project. Distribution and usage terms are subject to project agreement.
