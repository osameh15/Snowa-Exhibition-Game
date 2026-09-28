# Snowa Exhibition Gaming Platform — Technical Documentation (Phase 0)

Implementation-ready technical documentation derived from the approved product source
[`source/Snowa_Exhibition_Gaming_Platform_Product_Game_Design_Spec_v1.0.docx`](source/) (**SPEC**, v1.0, 28 Sep 2026). The source file is not modified.

Status: **Phase 0 draft — awaiting approval.** No implementation has started.

## Start here

1. [Technical overview](00-overview/01-technical-overview.md)
2. [Open questions & assumptions](12-planning/04-open-questions.md) — what still needs decisions
3. [ADR index](11-decisions/README.md) — proposed architecture decisions
4. [Phase 1 plan](12-planning/01-phase-1-implementation-plan.md)

## Index

### 00 — Overview
- [Technical overview](00-overview/01-technical-overview.md)
- [Scope](00-overview/02-scope.md)
- [Glossary](00-overview/03-glossary.md)
- [Requirement traceability matrix](00-overview/04-requirement-traceability-matrix.md)
- [Source analysis notes](00-overview/05-source-analysis.md)

### 01 — Architecture
- [System context](01-architecture/01-system-context.md)
- [Containers & components](01-architecture/02-containers-and-components.md)
- [Client architecture](01-architecture/03-client-architecture.md)
- [Game runtime architecture](01-architecture/04-game-runtime-architecture.md)
- [Backend architecture](01-architecture/05-backend-architecture.md)
- [Admin architecture](01-architecture/06-admin-architecture.md)
- [Real-time architecture](01-architecture/07-realtime-architecture.md)
- [Integration architecture](01-architecture/08-integration-architecture.md)
- [Trust boundaries](01-architecture/09-trust-boundaries.md)
- [Deployment topology](01-architecture/10-deployment-topology.md)
- [Localization, RTL & Persian typography](01-architecture/11-localization-and-rtl.md)
- [PWA & mobile behavior](01-architecture/12-pwa-and-mobile.md)
- [Analytics & telemetry](01-architecture/13-analytics-and-telemetry.md)

### 02 — Domain
- [Domain model & state machine index](02-domain/01-domain-model.md)
- [Participant lifecycle, onboarding & OTP](02-domain/02-participant-lifecycle.md)
- [Game availability, session & attempt lifecycle](02-domain/03-game-session-attempt-lifecycle.md)
- [Best score & leaderboard rules](02-domain/04-best-score-and-leaderboard.md)
- [Raffle ticket model](02-domain/05-raffle-tickets.md)
- [Reward model](02-domain/06-rewards.md)
- [Live raffle model](02-domain/07-live-raffle.md)
- [Audit model](02-domain/08-audit-model.md)

### 03 — Data
- [Conceptual schema (ERD)](03-data/01-conceptual-schema.md)
- [Entity definitions](03-data/02-entity-definitions.md)
- [Invariants, transactions & idempotency](03-data/03-invariants-and-transactions.md)
- [Auditing, retention & data lifecycle](03-data/04-auditing-retention-lifecycle.md)

### 04 — API
- [API principles](04-api/01-api-principles.md)
- [Auth, OTP & profile API](04-api/02-auth-and-profile-api.md)
- [Game session & score submission API](04-api/03-game-session-api.md)
- [Lobby, leaderboard & rewards API](04-api/04-lobby-leaderboard-rewards-api.md)
- [Admin API](04-api/05-admin-api.md)
- [Raffle API](04-api/06-raffle-api.md)
- [Real-time event contracts](04-api/07-realtime-events.md)
- [External Snowa API adapter contract](04-api/08-external-snowa-adapter.md)

### 05 — Games
- [Shared game runtime](05-games/01-shared-game-runtime.md)
- [Score validation framework](05-games/02-score-validation-framework.md)
- [Spin Perfect](05-games/03-spin-perfect.md)
- [Fridge Rush](05-games/04-fridge-rush.md)
- [Vision Hunt](05-games/05-vision-hunt.md)

### 06 — Admin
- [Information architecture](06-admin/01-information-architecture.md)
- [Roles & permissions](06-admin/02-roles-and-permissions.md)
- [Game control & reward management](06-admin/03-game-and-reward-control.md)
- [Participants & audit history](06-admin/04-participants-and-audit.md)
- [Live operations (leaderboard, scores, raffle, displays, integrations)](06-admin/05-live-operations.md)
- [Reports](06-admin/06-reports.md)
- [Operator safety UX](06-admin/07-operator-safety-ux.md)

### 07 — Security
- [Threat model](07-security/01-threat-model.md)
- [Authentication, OTP abuse & sessions](07-security/02-authentication-and-sessions.md)
- [Anti-cheat & score integrity](07-security/03-anti-cheat-and-score-integrity.md)
- [Rate limiting & abuse](07-security/04-rate-limiting-and-abuse.md)
- [Raffle & reward integrity](07-security/05-raffle-and-reward-integrity.md)
- [API & transport security (+ audit requirements)](07-security/06-api-and-transport-security.md)
- [Privacy & data protection](07-security/07-privacy-and-data-protection.md)

### 08 — Quality
- [Testing strategy](08-quality/01-testing-strategy.md)
- [Device & browser matrix](08-quality/02-device-browser-matrix.md)
- [Accessibility & localization testing](08-quality/03-accessibility-and-localization-testing.md)
- [Load testing & failure injection](08-quality/04-load-and-resilience-testing.md)
- [Raffle & reward tests](08-quality/05-raffle-and-reward-tests.md)
- [Acceptance strategy](08-quality/06-acceptance-strategy.md)

### 09 — Operations
- [Environments, configuration & secrets](09-operations/01-environments-and-configuration.md)
- [Deployment & rollback](09-operations/02-deployment-and-rollback.md)
- [Observability](09-operations/03-observability.md)
- [Event-day runbook](09-operations/04-event-day-runbook.md)
- [Degraded modes](09-operations/05-degraded-modes.md)
- [Backup, recovery & reconciliation](09-operations/06-backup-recovery-and-reconciliation.md)

### 10 — Performance
- [Performance budgets](10-performance/01-performance-budgets.md)
- [Caching, lazy loading & network resiliency](10-performance/02-caching-loading-network.md)
- [Real-time & leaderboard performance](10-performance/03-realtime-and-leaderboard-performance.md)

### 11 — Decisions
- [ADR index](11-decisions/README.md) (ADR-001 … ADR-012)

### 12 — Planning
- [Phase 1 implementation plan](12-planning/01-phase-1-implementation-plan.md)
- [Milestones & dependency map](12-planning/02-milestones-and-dependencies.md)
- [Technical risk register](12-planning/03-risk-register.md)
- [Open questions & assumptions](12-planning/04-open-questions.md)
- [Definition of ready](12-planning/05-definition-of-ready.md)

## Required diagrams — where to find them

| Diagram | Location |
|---|---|
| First-time participant sequence | [Participant lifecycle §4](02-domain/02-participant-lifecycle.md#4-sequence--first-time-participant) |
| Returning participant sequence | [Participant lifecycle §5](02-domain/02-participant-lifecycle.md#5-sequence--returning-participant) |
| Game attempt sequence | [Lifecycle §5](02-domain/03-game-session-attempt-lifecycle.md#5-sequence--game-attempt-full-pipeline) |
| Retry attempt sequence | [Lifecycle §6](02-domain/03-game-session-attempt-lifecycle.md#6-sequence--retry-attempt-existing-best-score) |
| Live leaderboard sequence | [Leaderboard §4](02-domain/04-best-score-and-leaderboard.md#4-live-leaderboard-update-flow) |
| Base raffle ticket sequence | [Raffle tickets §4](02-domain/05-raffle-tickets.md#4-base-ticket-grant-sequence-idempotency) |
| Dynamic reward sequence | [Rewards §6](02-domain/06-rewards.md#6-dynamic-reward-sequence) |
| Live raffle sequence | [Live raffle §5](02-domain/07-live-raffle.md#5-live-raffle-sequence) |
| External API delivery sequence | [Snowa adapter §6](04-api/08-external-snowa-adapter.md#6-external-api-delivery-sequence) |
| State machines (onboarding, OTP, game availability, session, attempt, reward grant, ticket, draw, delivery, event) | [State machine index](02-domain/01-domain-model.md#5-state-machine-index) |
| Architecture (context, containers, modules, trust, deployment, real-time fan-out) | `01-architecture/` |

## Conventions

MUST/SHOULD/MAY per RFC 2119 · `[SPEC §x]` cites the source · OQ-nn = open question · A-nn = assumption · ADR-nnn = architecture decision · INV-nn = data invariant · R-nn = risk.
