# Architecture Decision Records

Format: Context → Options → Decision → Consequences. Status values: **Accepted** (matches approved baseline), **Proposed** (recommendation awaiting approval), **Superseded**.

Architecture decisions (this folder) are separate from **product decisions**, which are tracked as open questions in [Open questions](../12-planning/04-open-questions.md).

| ADR | Title | Status | Blocks Phase 1? |
|---|---|---|---|
| [ADR-001](ADR-001-frontend-architecture.md) | Frontend: Nuxt 4 SPA, separate participant/admin apps | Accepted (baseline) with proposed refinements | — |
| [ADR-002](ADR-002-backend-technology.md) | Backend: Node.js + TypeScript + Fastify modular monolith | Proposed | **Yes** |
| [ADR-003](ADR-003-realtime-transport.md) | Real-time: SSE + REST snapshots (+ polling fallback) | Proposed | No (Phase 2) |
| [ADR-004](ADR-004-persistence.md) | PostgreSQL as single stateful dependency; Redis deferred | Proposed | **Yes** |
| [ADR-005](ADR-005-otp-provider-boundary.md) | OTP provider behind `OtpSender` port | Proposed | No (fake sender in dev) — vendor needed before staging |
| [ADR-006](ADR-006-game-runtime.md) | Phaser 3 gameplay + shared deterministic game-core | Accepted (baseline) with proposed refinements | — |
| [ADR-007](ADR-007-reward-architecture.md) | Core ticket fixed step + data-driven extra reward engine | Proposed | No (Phase 2) |
| [ADR-008](ADR-008-external-integration.md) | Transactional outbox + Snowa adapter | Proposed | No (outbox skeleton in Phase 1) |
| [ADR-009](ADR-009-authentication-sessions.md) | Cookie-based opaque sessions; admin TOTP | Proposed | **Yes** |
| [ADR-010](ADR-010-score-validation.md) | Server replay of action logs with shared deterministic code | Proposed | **Yes** |
| [ADR-011](ADR-011-raffle-selection.md) | Weighted selection without replacement, recorded CSPRNG seed | Proposed | No (Phase 5) |
| [ADR-012](ADR-012-deployment-topology.md) | Containers on small VM footprint; managed PostgreSQL preferred | Proposed | No (staging needed by M1 end) |
