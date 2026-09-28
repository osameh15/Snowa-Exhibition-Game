# Technical Risk Register

Likelihood/Impact: L/M/H.

| ID | Risk | L | I | Mitigation | Owner | Trigger / indicator |
|---|---|---|---|---|---|---|
| R-01 | Venue mobile network congestion makes onboarding/game loading slow | H | H | Small shell, SW caching, lazy game assets, booth Wi-Fi option, late-submission window, on-site network test at T-1 | DevOps + Ops | Field test RTT/throughput |
| R-02 | OTP SMS delayed/failing on event day | M | H | Vendor evaluation, delivery monitoring, optional secondary vendor, clear Persian retry UX, runbook | Backend + Snowa | OTP success < 90 % |
| R-03 | Snowa API contract arrives late or changes | H | M | Adapter + outbox + fake receiver; delivery backlog tolerable; Phase 6 buffer | Backend + Snowa IT | No contract by M4 |
| R-04 | Client/server scoring drift (engine float differences, version skew) causes false rejections | M | H | Integer math, shared game-core, cross-engine golden tests, config checksum, `426` for old runtimes, `SCORE_MISMATCH` alert | Game + Backend | Any mismatch in QA/production |
| R-05 | Duplicate records in Snowa after ambiguous timeouts (no idempotency support) | M | M | Request idempotency key; per-pair ordering; supersede mode; reconciliation | Backend | Contract lacks idempotency |
| R-06 | Bots/cheaters occupy top ranks | M | M | Replay validation, plausibility flags, top-rank review before prizes, invalidation tooling | Ops + Backend | Flag rate, suspicious top scores |
| R-07 | iOS Safari quirks (audio unlock, viewport, WebGL memory, standalone cookie jar) | H | M | Early device testing each milestone; fallbacks; memory budgets | Frontend | Crashes/tab reloads on iOS |
| R-08 | Low-end Android can't hold 60 FPS | M | M | Budgets, atlases, pooled objects, VFX quality tiers | Game | FPS p5 < 45 |
| R-09 | Hosting constraints (foreign services blocked, no managed DB) | M | H | Self-hosted deps, provider-agnostic containers, early decision (OQ-02) | DevOps | No hosting decision by M1 end |
| R-10 | Art assets late or too heavy | M | M | Placeholder art pipeline, budgets enforced in CI (asset size check), game-by-game production order [SPEC §26] | Art + Game | Asset > budget |
| R-11 | Persian font licensing late | M | L | Placeholder open-license font; swap via tokens | Brand | No license by M6 |
| R-12 | Raffle rules undefined/contested | M | H | Legal input (OQ-29), reproducible draws, audit, void procedure | Legal + Ops | No rules by M5 |
| R-13 | Operator error in live controls (wrong attempt limit, unlimited reward, wrong draw) | M | H | Tiered confirmations, reasons, impact previews, audit, training at rehearsal | Ops + Frontend | Rehearsal mistakes |
| R-14 | Reward misconfiguration overspends inventory | L | H | Server-side safety rules, atomic caps, low-inventory alerts | Backend | — |
| R-15 | Real scale exceeds assumptions | M | H | Load test ×5, upgrade paths (replicas, Redis), capacity headroom | DevOps | OQ-05 answer |
| R-16 | Single PostgreSQL failure | L | H | Managed HA / replica + PITR, restore drill, late window for clients | DevOps | — |
| R-17 | Offensive names on public display | M | M | Normalization, deny-list, hide action, `NAME_INITIAL` policy option | Ops | First incident |
| R-18 | Schedule compression across 7 phases before event date (unknown, OQ-22) | M | H | Vertical slice first; shared runtime reuse; parallel art; scope guard (no combined leaderboard, no profile edit) | PM | Milestone slip |
| R-19 | In-app browsers (QR scanners in messaging apps) behave differently (storage, WebGL, OTP autofill) | M | M | Test matrix includes in-app browsers; "open in browser" hint if unsupported | Frontend | QA findings |
| R-20 | Attempt-consumption policy perceived as unfair (refresh loses attempt) | M | M | Clear pre-game notice, pause support, operator bonus attempts, OQ-15 sign-off | Product | Booth complaints |
