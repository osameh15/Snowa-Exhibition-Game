# Definition of Ready

## 1. Phase 0 exit criteria (status at documentation hand-off)

| # | Criterion | Status | Evidence |
|---|---|---|---|
| 1 | Entire product source analyzed | Done | [Source analysis](../00-overview/05-source-analysis.md) |
| 2 | Documentation internally consistent | Done (self-reviewed; cross-links checked) | this set |
| 3 | Architecture responsibilities clear | Done | [Containers & components](../01-architecture/02-containers-and-components.md) ownership matrix |
| 4 | Trust boundaries explicit | Done | [Trust boundaries](../01-architecture/09-trust-boundaries.md) |
| 5 | Core entities & invariants documented | Done | [Domain model](../02-domain/01-domain-model.md), [Invariants](../03-data/03-invariants-and-transactions.md) |
| 6 | API boundaries specified | Done | `04-api/` |
| 7 | Three games implementation-ready | Done (values tunable in prototype) | `05-games/` |
| 8 | Admin capabilities specified | Done | `06-admin/` |
| 9 | Best Score unambiguous | Done | [Best score & leaderboard](../02-domain/04-best-score-and-leaderboard.md) |
| 10 | Raffle Ticket unambiguous | Done (weighting pending OQ-08) | [Raffle tickets](../02-domain/05-raffle-tickets.md) |
| 11 | Live Raffle architecture | Done | [Live raffle](../02-domain/07-live-raffle.md) |
| 12 | Dynamic Reward architecture | Done | [Rewards](../02-domain/06-rewards.md) |
| 13 | External API boundary | Done (contract TBD) | [Snowa adapter](../04-api/08-external-snowa-adapter.md) |
| 14 | PWA/mobile constraints | Done | [PWA & mobile](../01-architecture/12-pwa-and-mobile.md) |
| 15 | Security & anti-cheat strategy | Done | `07-security/` |
| 16 | Testing strategy | Done | `08-quality/` |
| 17 | Operations/runbook architecture | Done | `09-operations/` |
| 18 | Architecture vs product decisions separated | Done | ADRs vs [Open questions](04-open-questions.md) |
| 19 | Phase 1 plan exists | Done | [Phase 1 plan](01-phase-1-implementation-plan.md) |
| 20 | No production implementation started | Done | repository contains only `docs/` |

## 2. Ready to start Phase 1 when

- [ ] ADR-002 (backend) approved (OQ-01).
- [ ] ADR-004 (PostgreSQL) and ADR-009 (sessions) approved.
- [ ] ADR-010 (replay validation) approved.
- [ ] OQ-15 attempt/interruption policy accepted by product (defaults may be tuned later).
- [ ] Hosting direction for staging agreed (OQ-02) — or explicit approval to start with local/compose environment.
- [ ] Spin Perfect placeholder art direction confirmed (final art may follow).

## 3. Definition of Ready for an implementation story

A story is ready when it has: linked requirement IDs (SPEC/AC/INV), linked doc section, API contract (if any), data changes identified, acceptance tests listed, open questions affecting it resolved or defaulted, Persian copy keys identified, and analytics events named.

## 4. Definition of Done (per story)

Code + tests (unit/integration/E2E as applicable) green; RTL/localization lint clean; docs updated if behavior differs; metrics/logs added for new flows; no new un-triaged security findings; reviewed.
