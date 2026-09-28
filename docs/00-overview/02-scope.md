# Scope

## 1. In scope for the exhibition release (v1.0)

| Area | Included | SPEC |
|---|---|---|
| Entry | QR-first mobile browser entry; optional PWA install | §2 PD-01, §20 |
| Identity | Iranian mobile number + OTP; display name on first verification | §6 |
| Lobby | Greeting, ticket progress, three stable game cards with state, attempts, best score, rank | §7 |
| Games | Spin Perfect, Fridge Rush, Vision Hunt on a shared runtime | §8–11 |
| Scoring | Best valid score per participant per game; attempt history retained | §12 |
| Leaderboards | Independent per-game live leaderboards; public display mode | §14 |
| Rewards | Fixed base raffle ticket per game (first valid completion); admin-configured extra rewards (discount code, extra tickets, physical prize, benefit, special) | §13 |
| Live raffle | Filter builder, eligible-count preview, server-side selection, immutable draw records, public reveal | §15 |
| Admin | Dashboard, game control, attempt limits, rewards, participants, leaderboards, raffle, reports, audit, integration monitoring | §16 |
| External API | Server-to-server delivery of accepted results via outbox and adapter | §17 |
| Real-time | Live admin feed, leaderboards, display, draw presentation | §19 |
| Analytics | First-party product events + operational telemetry | §23 |
| Localization | Persian/RTL participant UI; Persian/RTL admin (bilingual admin is OQ) | §5 |

## 2. Explicitly out of scope (v1.0)

| Item | Reason / SPEC |
|---|---|
| Combined / event-wide leaderboard | §12.4 — MUST NOT be invented |
| Participant profile editing | §6.3, §32 — open item; not built unless approved (OQ-09) |
| Non-Iranian phone numbers | §6.1 — primary format is Iranian mobile; expansion requires approval |
| Native apps / app store distribution | §2 PD-01 |
| Offline gameplay without prior session authorization | §20.2, §21 — a session requires the server |
| General-purpose loyalty platform features (points wallet, social/friends) | §1 — campaign product. Note: concept art shows a "friends" tab on leaderboard; not in SPEC text, therefore excluded (see [Source analysis](05-source-analysis.md)) |
| Payment, e-commerce redemption integration | Not in SPEC; discount codes are displayed only |
| Third-party analytics carrying PII | §23 — not without explicit approval |
| English participant UI in production | §5 |

## 3. Phase 0 scope (this deliverable)

Documentation and architecture only. No production code, no scaffolding. Phase 0 exit criteria are tracked in [Definition of Ready](../12-planning/05-definition-of-ready.md).

## 4. Delivery phases (from SPEC §31, technically refined)

| Phase | Deliverable | Technical focus |
|---|---|---|
| 0 | This documentation set | Remove ambiguity |
| 1 | Vertical slice: OTP → name → lobby → Spin Perfect → result → best score → base ticket → leaderboard | Platform loop, session protocol, replay validation, outbox skeleton |
| 2 | Shared platform: admin game controls, reward engine, participants, real-time leaderboard | SSE, reward engine, RBAC, audit |
| 3 | Fridge Rush | Drag/drop runtime, touch handling |
| 4 | Vision Hunt | Seeded layouts, reaction validation |
| 5 | Live raffle | Snapshot, selection, reveal protocol |
| 6 | External integration hardening | Final Snowa contract, retries, reconciliation |
| 7 | Exhibition hardening | Device matrix, load, anti-cheat tuning, rehearsal |

Detailed plan: [Phase 1 plan](../12-planning/01-phase-1-implementation-plan.md), [Milestones](../12-planning/02-milestones-and-dependencies.md).

## 5. Scale assumptions (A-01, pending OQ-05)

No expected-scale figures exist in SPEC (§32 lists this as open). To make design choices concrete, this package assumes:

| Metric | Assumed design value | Load-test target (×3–5) |
|---|---|---|
| Total verified participants per event day | 10,000 | 50,000 |
| Peak concurrent active participants | 500 | 2,500 |
| Peak result submissions | 20/s | 100/s |
| Peak OTP requests | 10/s | 50/s |
| Admin + public display connections | ≤ 30 | 100 |
| Event duration | 1–5 days | — |

These are **assumptions**, not measurements: infrastructure MUST NOT be sized from them alone. Phase 1 includes load tests on the staging VPS before final production sizing. If real figures exceed the load-test targets, the [incremental scaling path](../01-architecture/10-deployment-topology.md#5-incremental-scaling-path-future-options-not-v1-requirements), [ADR-004](../11-decisions/ADR-004-persistence.md) and [ADR-003](../11-decisions/ADR-003-realtime-transport.md) define the upgrade paths (larger VPS, separate PostgreSQL, separate worker, more Fastify instances, Redis cache/pub-sub).
