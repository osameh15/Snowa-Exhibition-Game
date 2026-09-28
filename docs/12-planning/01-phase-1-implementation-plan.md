# Phase 1 Implementation Plan — Vertical Slice

Goal [SPEC §31]: validate the entire platform loop with the simplest game:
**OTP → name → lobby → Spin Perfect → result → best score → base ticket → leaderboard**, deployed to staging, playable on real phones.

Not started. Awaiting Phase 0 approval.

## 1. In scope / out of scope

| In scope | Out of scope (later milestones) |
|---|---|
| Monorepo, CI, lint rules (RTL logical CSS, no hard-coded strings, game-core determinism rules) | Admin app (M2) — Phase 1 uses seed scripts/SQL for game settings |
| PostgreSQL schema for identity, catalog, play, progress, tickets, outbox, audit (core), idempotency | Reward engine, code pools (M2) |
| OTP flow with fake sender + real vendor adapter in staging (if OQ-03 answered) | SSE real-time (M2); leaderboard via REST |
| Name onboarding, lobby with 3 cards (2 shown as "coming soon/unavailable" via DISABLED state) | Fridge Rush, Vision Hunt (M3, M4) |
| Session protocol (issue/start/result/get/abandon), sweeper job | Raffle (M5) |
| game-core: PRNG, Spin Perfect simulation/scoring/replay/bounds, golden tests | Real Snowa adapter (M6) |
| Spin Perfect Phaser scene, shared GameHost, pause/background policy, pending-result recovery | Public display (M2) |
| Result pipeline: validation, best score, base ticket, outbox + fake receiver worker | |
| Result screen, participant leaderboard, profile summary (basic) | |
| Service worker/app shell caching; asset pipeline for Spin Perfect | |
| Staging deployment (proxy, api×2, worker, PostgreSQL) + basic metrics/logs | |

## 2. Recommended first milestone: **M1.1 Walking Skeleton** (≈ 2 weeks)

Thin end-to-end path with a placeholder "tap counter" game running on the real protocol, to de-risk infrastructure and the session/validation pipeline before investing in art-heavy gameplay.

| # | Deliverable | Done when |
|---|---|---|
| 1 | Monorepo (`apps/participant`, `apps/server`, `packages/contracts`, `packages/game-core`, `packages/i18n-fa`), CI with typecheck/lint/unit/integration (Testcontainers) | Green pipeline on empty features |
| 2 | Migrations: participants, otp_challenges, participant_sessions, events, games, game_settings, game_config_versions, game_sessions, attempts, attempt_payloads, attempt_flags, participant_game_progress, raffle_tickets, external_deliveries(+attempts), audit_log, idempotency_keys, rate_limit_buckets | Constraints from [entity definitions](../03-data/02-entity-definitions.md) present; invariant tests pass |
| 3 | Auth API + Persian screens (phone, OTP, name) with fake sender | AC-001/002 E2E green locally |
| 4 | Lobby API + UI with 3 stable cards | AC-004 (structure) |
| 5 | Session protocol + placeholder game via GameModule contract, replay validation of a trivial deterministic game | Concurrency tests (INV-02..05) green |
| 6 | Result pipeline: best score, base ticket, outbox → fake receiver | AC-007/009/019 integration tests green |
| 7 | Staging environment live (or local compose if OQ-02 pending) | Team can play on phones |

## 3. M1.2 Spin Perfect (≈ 2–3 weeks)

- game-core Spin Perfect: θ(t), zones, classification, combo, scoring, pass-through, cooldown, maxScore, plausibility stats; golden logs from SPEC examples; property tests; cross-engine test.
- Phaser scene + critical/deferred asset groups (placeholder art first, final art when delivered).
- HUD overlay, countdown, pause/background, back-guard, context loss handling.
- Calibration build: collect tester logs → set bounds/flags → publish config version `1.0.0`.

## 4. M1.3 Result, leaderboard, hardening (≈ 1 week)

- Result screen per SPEC §13.5 (base ticket granted vs active).
- Participant leaderboard (keyset API, public projection, own row).
- Refresh/crash recovery flows; offline gameplay submission.
- Device smoke on tier A matrix; performance budget check.

## 5. Phase 1 acceptance

AC-001, 002, 003 (slice screens), 004 (structure), 006, 007, 008, 009, 011, 012, 019, 020, 021 (Spin Perfect) verified; load smoke (L2 at 20 results/s) passes on staging; no `SCORE_MISMATCH` across device matrix.

## 6. Team allocation (suggested)

| Stream | People | Focus |
|---|---|---|
| Backend | 1–2 | schema, auth, session/result pipeline, outbox, CI, staging |
| Frontend | 1 | shell, auth/lobby/result/leaderboard UI, RTL, SW |
| Game | 1 | game-core + Phaser Spin Perfect, GameHost |
| QA | 1 (from M1.2) | E2E, device matrix |
| DevOps | part-time | staging, observability basics |
