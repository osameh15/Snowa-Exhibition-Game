# Architecture Decision Records

Format: Context → Options → Decision → Consequences. Status values: **Accepted**, **Proposed** (recommendation awaiting approval; does not block Phase 1 unless stated), **Superseded**.

Architecture decisions (this folder) are separate from **product decisions**, which are tracked in [Open questions](../12-planning/04-open-questions.md).

Guiding constraint (final Phase 0 correction): the v1 exhibition deployment is right-sized to one VPS, while score integrity, anti-cheat, authentication, admin security, raffle and reward integrity, auditability and result durability are **not** simplified.

| ADR | Title | Status | Blocks Phase 1? |
|---|---|---|---|
| [ADR-001](ADR-001-frontend-architecture.md) | Frontend: one Nuxt 4 SPA/PWA with participant, `/admin`, `/display` routes | Accepted | — |
| [ADR-002](ADR-002-backend-technology.md) | Backend: Node.js 22 + TypeScript + Fastify, one modular-monolith process with in-process jobs | **Accepted** | — |
| [ADR-003](ADR-003-realtime-transport.md) | Real-time: REST + SSE + polling fallback, in-process bus | Accepted | — |
| [ADR-004](ADR-004-persistence.md) | PostgreSQL 16+ as the only durable/stateful service; Redis deferred | **Accepted** | — |
| [ADR-005](ADR-005-otp-provider-boundary.md) | OTP provider behind `OtpSender` port | Proposed (vendor TBD) | No (fake sender until vendor chosen) |
| [ADR-006](ADR-006-game-runtime.md) | Phaser 3 gameplay + shared deterministic game-core | Accepted | — |
| [ADR-007](ADR-007-reward-architecture.md) | Core ticket fixed step + data-driven extra reward engine | Proposed | No (Phase 2) |
| [ADR-008](ADR-008-external-integration.md) | PostgreSQL transactional outbox + adapter; sender runs as in-process job | Accepted | — |
| [ADR-009](ADR-009-authentication-sessions.md) | Server-managed opaque sessions; separate participant/admin/display namespaces; admin Argon2id + TOTP | **Accepted** | — |
| [ADR-010](ADR-010-score-validation.md) | Deterministic server-side score replay | **Accepted** | — |
| [ADR-011](ADR-011-raffle-selection.md) | Weighted selection without replacement, recorded CSPRNG seed | Proposed (weighting semantics OQ-08) | No (Phase 5) |
| [ADR-012](ADR-012-deployment-topology.md) | Single VPS, systemd, no Docker; incremental scaling path | Accepted (vendor TBD) | — |
| [ADR-013](ADR-013-reuse-by-configuration.md) | Reuse by configuration and modules, not multi-tenancy | Accepted | — |
