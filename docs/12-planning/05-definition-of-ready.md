# Definition of Ready

## 1. Phase 0 exit criteria

| # | Criterion | Status | Evidence |
|---|---|---|---|
| 1 | Entire product source analyzed | Done | [Source analysis](../00-overview/05-source-analysis.md) |
| 2 | Documentation internally consistent | Done (re-reviewed after the final architecture correction; links and diagrams validated) | this set |
| 3 | Architecture responsibilities clear | Done | [Runtime units & modules](../01-architecture/02-containers-and-components.md) |
| 4 | Trust boundaries explicit | Done | [Trust boundaries](../01-architecture/09-trust-boundaries.md) |
| 5 | Core entities & invariants documented | Done | [Domain model](../02-domain/01-domain-model.md), [Invariants](../03-data/03-invariants-and-transactions.md) |
| 6 | API boundaries specified | Done | `04-api/` |
| 7 | Three games implementation-ready | Done (values tunable in prototype) | `05-games/` |
| 8 | Admin capabilities specified | Done | `06-admin/`, [Admin security baseline](../01-architecture/06-admin-architecture.md#1a-admin-security-baseline-non-negotiable) |
| 9 | Best Score unambiguous | Done | [Best score & leaderboard](../02-domain/04-best-score-and-leaderboard.md) |
| 10 | Raffle Ticket unambiguous | Done (weighting pending OQ-08, Phase 5) | [Raffle tickets](../02-domain/05-raffle-tickets.md) |
| 11 | Live Raffle architecture | Done | [Live raffle](../02-domain/07-live-raffle.md) |
| 12 | Dynamic Reward architecture | Done | [Rewards](../02-domain/06-rewards.md) |
| 13 | External API boundary | Done (contract TBD) | [Snowa adapter](../04-api/08-external-snowa-adapter.md) |
| 14 | PWA/mobile constraints | Done | [PWA & mobile](../01-architecture/12-pwa-and-mobile.md) |
| 15 | Security & anti-cheat strategy | Done (unchanged by right-sizing) | `07-security/`, [ADR-010](../11-decisions/ADR-010-score-validation.md) |
| 16 | Testing strategy | Done | `08-quality/` |
| 17 | Operations/runbook architecture | Done (single-VPS, systemd) | `09-operations/` |
| 18 | Architecture vs product decisions separated | Done | ADRs vs [Open questions](04-open-questions.md) |
| 19 | Phase 1 plan exists | Done | [Phase 1 plan](01-phase-1-implementation-plan.md) |
| 20 | No production implementation started | Done | repository contains only documentation |
| 21 | Deployment right-sized with incremental scaling path | Done | [Deployment topology](../01-architecture/10-deployment-topology.md) |
| 22 | Future brand reuse approach defined | Done | [Reuse & branding](../01-architecture/14-reuse-and-branding.md), [ADR-013](../11-decisions/ADR-013-reuse-by-configuration.md) |

## 2. Ready to start Phase 1 when

- [x] ADR-002 (backend) accepted — OQ-01 resolved.
- [x] ADR-004 (PostgreSQL), ADR-009 (sessions) and ADR-010 (deterministic replay) accepted.
- [x] OQ-15 attempt/interruption policy approved.
- [x] Staging direction agreed (OQ-02): single VPS, `https://snowa-games.osameh.dev`.
- [ ] Architecture review of the final Phase 0 correction completed (the only remaining gate).
- [ ] Staging VPS provisioned (vendor TBD) — needed by the end of M1.1, not to start coding (local development first).

Placeholder Spin Perfect art is sufficient to start; final art may follow.

## 3. Definition of Ready for an implementation story

A story is ready when it has: linked requirement IDs (SPEC/AC/INV), linked doc section, API contract (if any), data changes identified, acceptance tests listed, open questions affecting it resolved or defaulted, Persian copy keys identified, and analytics events named.

## 4. Definition of Done (per story)

Code + tests (unit/integration/E2E as applicable) green; RTL/localization lint clean; determinism lint clean for `game-core`; docs updated if behavior differs; metrics/logs added for new flows; no new un-triaged security findings; reviewed.
