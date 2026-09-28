# Phase 1 Implementation Plan — Vertical Slice

Goal [SPEC §31]: validate the entire platform loop with the simplest game:
**OTP → name → lobby → Spin Perfect → result → best score → base ticket → leaderboard**, deployed to the staging VPS (`https://snowa-games.osameh.dev`), playable on real phones, with load measurements that inform final production sizing.

Not started. Awaiting architecture review of the final Phase 0 correction.

## 1. In scope / out of scope

| In scope | Out of scope (later milestones) |
|---|---|
| Monorepo (`apps/web`, `apps/server`, `packages/*`, `brands/snowa`), CI, lint rules (RTL logical CSS, no hard-coded strings, `game-core` determinism, module-boundary imports, no brand literals outside `brands/`) | Admin control-room screens beyond minimal login (M2) — Phase 1 uses seed scripts for game settings |
| PostgreSQL schema for identity, catalog, play, progress, tickets, outbox, audit (core), idempotency, rate limits | Reward engine, code pools (M2) |
| OTP flow with fake sender + real vendor adapter in staging (if OQ-03 answered) | Admin/display SSE views (M2); participant leaderboard via REST |
| Name onboarding, lobby with 3 cards (2 shown unavailable via DISABLED state) | Fridge Rush, Vision Hunt (M3, M4) |
| Session protocol (issue/start/result/get/abandon) with the approved attempt policy; sweeper job | Raffle (M5) |
| `game-core`: PRNG, Spin Perfect simulation/scoring/replay/bounds, golden + cross-engine tests | Real Snowa adapter (M6) |
| Spin Perfect Phaser scene, shared GameHost, pause/background policy, pending-result recovery | Public display (M2) |
| Result pipeline: validation, best score, base ticket, outbox + in-process sender job → fake receiver | |
| Result screen, participant leaderboard, profile summary (basic) | |
| Service worker/app shell caching; asset pipeline for Spin Perfect | |
| Staging VPS setup: Ubuntu 24.04, Node.js 22, PostgreSQL 16+, Nginx or Caddy + Let's Encrypt, systemd unit, service user, firewall, daily off-server backup, uptime check | |
| Load measurement (L1/L2 profiles) on the staging VPS; sizing report for production | |

## 2. First milestone: **M1.1 Walking Skeleton** (≈ 2 weeks)

Thin end-to-end path with a placeholder "tap counter" game on the real protocol, to de-risk deployment and the session/validation pipeline before art-heavy gameplay.

| # | Deliverable | Done when |
|---|---|---|
| 1 | Monorepo, CI (typecheck, lint, unit, integration with PostgreSQL in CI) | Green pipeline |
| 2 | Migrations for core tables ([entity definitions](../03-data/02-entity-definitions.md)) | Constraints present; invariant tests pass |
| 3 | Auth API + Persian screens (phone, OTP, name) with fake sender; three session namespaces wired (participant only exposed) | AC-001/002 E2E green locally |
| 4 | Lobby API + UI with 3 stable cards | AC-004 structure |
| 5 | Session protocol + placeholder deterministic game via GameModule contract + server replay | Concurrency tests (INV-02..05) green |
| 6 | Result pipeline: best score, base ticket, outbox → in-process sender → fake receiver; restart safety | AC-007/009/019 integration tests; process kill test keeps deliveries |
| 7 | Staging VPS provisioned and deployed via release archive + systemd | Team plays on phones at `https://snowa-games.osameh.dev` |

## 3. M1.2 Spin Perfect (≈ 2–3 weeks)

- `game-core` Spin Perfect: θ(t), zones, classification, combo, scoring, pass-through, cooldown, maxScore, plausibility stats; golden logs from SPEC examples; property tests; cross-engine test.
- Phaser scene + critical/deferred asset groups (placeholder art first).
- HUD overlay, countdown, pause/background, back-guard, context-loss handling.
- Calibration build: collect tester logs → set bounds/flags → publish config version `1.0.0`.

## 4. M1.3 Result, leaderboard, measurement (≈ 1 week)

- Result screen per SPEC §13.5 (base ticket granted vs active).
- Participant leaderboard (keyset API, public projection, own row).
- Refresh/crash recovery flows; offline gameplay submission; late-result window.
- Device smoke on tier A matrix; performance budget check.
- Load profiles L1/L2 on the 1 vCPU / 2 GB staging VPS; record CPU, memory, p95 latency; extrapolate and recommend production size (starting point 2 vCPU / 4 GB).

## 5. Phase 1 acceptance

AC-001, 002, 003 (slice screens), 004 (structure), 006, 007, 008, 009, 011, 012, 019, 020, 021 (Spin Perfect) verified on staging; no `SCORE_MISMATCH` across the device matrix; load measurement report delivered; restart/kill tests show no lost accepted results or deliveries.

## 6. Team allocation (suggested)

| Stream | People | Focus |
|---|---|---|
| Backend | 1–2 | schema, auth, session/result pipeline, outbox + jobs, CI, staging VPS |
| Frontend | 1 | web app shell, auth/lobby/result/leaderboard UI, RTL, SW |
| Game | 1 | `game-core` + Phaser Spin Perfect, GameHost |
| QA | 1 (from M1.2) | E2E, device matrix, load runs |
