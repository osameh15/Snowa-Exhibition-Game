# Requirement Traceability Matrix

Links approved product requirements (SPEC decision IDs PD-xx, SPEC Appendix B IDs, and additional SPEC rules) to the responsible component, technical documents, API/domain entities and planned tests. Acceptance criteria AC-001…AC-022 are mapped to verification in [Acceptance strategy](../08-quality/06-acceptance-strategy.md).

Test ID prefixes: `U` unit, `GL` game-logic golden/property, `I` backend integration, `E` E2E, `D` device matrix, `L`/`F` load/failure ([Load & resilience](../08-quality/04-load-and-resilience-testing.md)), `RF`/`RW` raffle/reward ([tests](../08-quality/05-raffle-and-reward-tests.md)).

## 1. Product decisions (SPEC §2)

| Req | Requirement | Component | Documents | API / Entity | Tests | AC |
|---|---|---|---|---|---|---|
| PD-01 | QR-first, no install | participant-app, SW | [PWA & mobile](../01-architecture/12-pwa-and-mobile.md), [Client](../01-architecture/03-client-architecture.md) | `/?src=` landing | E (no-install flow), D | AC-020 |
| PD-02 | Phone + OTP required | identity module, `OtpSender` | [Participant lifecycle](../02-domain/02-participant-lifecycle.md), [Auth API](../04-api/02-auth-and-profile-api.md), [Auth security](../07-security/02-authentication-and-sessions.md) | `POST /auth/otp/*`; `otp_challenges`, `participant_sessions` | U (normalize), I (limits, lockout), E | AC-001 |
| PD-03 | Name after OTP for first-time users; returning skip | identity module, onboarding route guard | Participant lifecycle §2/§4/§5 | `PUT /me/profile`; `participants.display_name`; `403 PROFILE_INCOMPLETE` | E (skip attempt via URL), I | AC-001, AC-002 |
| PD-04 | Exactly three record games | catalog, game modules | `05-games/*` | `games` seed (3 rows) | E (lobby 3 cards) | AC-004 |
| PD-05 | Attempts admin-controlled per game, default 1 | catalog + play | [Lifecycle §1–4](../02-domain/03-game-session-attempt-lifecycle.md), [Game control](../06-admin/03-game-and-reward-control.md) | `game_settings.attempt_limit`; `PATCH /admin/.../attempt-limit`; T-START | I (concurrency start), E | AC-006, AC-013 |
| PD-06 | Best valid score wins | scoring module | [Best score](../02-domain/04-best-score-and-leaderboard.md), [Invariants](../03-data/03-invariants-and-transactions.md) INV-06 | `participant_game_progress.best_*` | I (6200→4100→7900), reconcile job test | AC-007 |
| PD-07 | One base ticket per participant per game | tickets module (result pipeline) | [Raffle tickets](../02-domain/05-raffle-tickets.md), ADR-007 | `raffle_tickets` partial UQ | I (duplicates, concurrency), RW-01 | AC-009 |
| PD-08 | Dynamic extra rewards without removing core | rewards module | [Rewards](../02-domain/06-rewards.md), [Reward mgmt](../06-admin/03-game-and-reward-control.md) | `reward_*` tables, admin reward API | RW-02…14 | AC-010, AC-014 |
| PD-09 | Server authority for attempts, scores, tickets, rewards, winners | api (all modules) | [Trust boundaries](../01-architecture/09-trust-boundaries.md), [Validation](../05-games/02-score-validation-framework.md), ADR-010 | result pipeline | GL, I, security tests | AC-016, AC-019 |
| PD-10 | Filterable live draw | raffle module, admin app, display | [Live raffle](../02-domain/07-live-raffle.md), [Raffle API](../04-api/06-raffle-api.md) | `draws`, `draw_entries`, `draw_winners` | RF-01…17 | AC-015, AC-016 |
| PD-11 | Persist first, then external API with retries | integration module, `jobs` outbox sender | [Snowa adapter](../04-api/08-external-snowa-adapter.md), ADR-008 | `external_deliveries` | I (outbox), F2 | AC-018 |
| PD-12 | Mobile-first portrait; tablet supported; desktop for admin/display | apps | [PWA & mobile](../01-architecture/12-pwa-and-mobile.md), [Device matrix](../08-quality/02-device-browser-matrix.md) | — | D | AC-021 |

## 2. SPEC Appendix B requirements

| Req | Requirement | Component | Documents | API / Entity | Tests |
|---|---|---|---|---|---|
| AUTH-001 | Phone + OTP required | identity | as PD-02 | as PD-02 | as PD-02 |
| AUTH-002 | Name after first OTP | identity | as PD-03 | as PD-03 | as PD-03 |
| LOC-001 | Persian/RTL participant UI | participant-app, i18n-fa | [Localization & RTL](../01-architecture/11-localization-and-rtl.md), [L10n testing](../08-quality/03-accessibility-and-localization-testing.md) | error codes → Persian copy | E (DOM scan), visual regression (AC-003) |
| GAME-001 | Three record games | game modules | [Spin Perfect](../05-games/03-spin-perfect.md), [Fridge Rush](../05-games/04-fridge-rush.md), [Vision Hunt](../05-games/05-vision-hunt.md) | `game_config_versions` | GL per game, D |
| GAME-002 | Attempt limit per game | catalog/play | as PD-05 | as PD-05 | as PD-05 |
| SCORE-001 | Official score = highest valid attempt | scoring | as PD-06 | as PD-06 | as PD-06 |
| SCORE-002 | All attempts retained for audit | play | [Entity defs](../03-data/02-entity-definitions.md) `attempts`, `attempt_payloads`; [Retention](../03-data/04-auditing-retention-lifecycle.md) | `GET /admin/attempts/{id}` | I (AC-008) |
| REW-001 | One base ticket per game first completion | tickets | as PD-07 | as PD-07 | as PD-07 |
| REW-002 | Dynamic extra rewards | rewards | as PD-08 | as PD-08 | as PD-08 |
| LB-001 | Live per-game leaderboard | scoring + realtime | [Best score & leaderboard](../02-domain/04-best-score-and-leaderboard.md), [Real-time](../01-architecture/07-realtime-architecture.md) | `GET /leaderboards/{slug}`, SSE `leaderboard_changed` | I (tie-break), L3, F13 (AC-012, AC-022) |
| RAFFLE-001 | Live server-side filtered raffle | raffle | as PD-10 | as PD-10 | as PD-10 |
| ADMIN-001 | Game enable/disable | catalog + admin | [Game control](../06-admin/03-game-and-reward-control.md), [Lifecycle §1](../02-domain/03-game-session-attempt-lifecycle.md) | `PATCH /admin/games/{slug}/state` | E (AC-005, AC-013) |
| ADMIN-002 | Change attempt limits | catalog + admin | as PD-05 | as PD-05 | as PD-05 |
| ADMIN-003 | Manage extra rewards | rewards + admin | as PD-08 | as PD-08 | as PD-08 |
| API-001 | Send phone/game/score after accepted result | integration | as PD-11 | canonical model | adapter contract tests |
| API-002 | Persist first, retry delivery | integration | as PD-11 | as PD-11 | F2, F4 |
| PWA-001 | Browser-first PWA; install optional | participant-app | as PD-01 | manifest, SW | E, D |
| SEC-001 | Server-authoritative session/result/reward/draw | api | as PD-09 | as PD-09 | as PD-09 |
| RT-001 | Near-real-time leaderboard/admin/public | realtime | [Real-time](../01-architecture/07-realtime-architecture.md), [Events](../04-api/07-realtime-events.md), ADR-003 | SSE channels | L3, L4, F3, F9, F13 |
| PERF-001 | Mobile performance; lazy game assets | participant-app, games | [Budgets](../10-performance/01-performance-budgets.md), [Caching](../10-performance/02-caching-loading-network.md) | asset manifests | D, CI bundle-size check |
| OPS-001 | Audit sensitive admin/draw actions | audit | [Audit model](../02-domain/08-audit-model.md) | `audit_log` | I (audit rows per action), RF-17 |

## 3. Additional SPEC rules

| SPEC | Requirement | Component | Documents | Entity / API | Tests |
|---|---|---|---|---|---|
| §3.1, AC-017 | Never expose full phone publicly | serializers | [Leaderboard §3](../02-domain/04-best-score-and-leaderboard.md#3-leaderboard-views), [Privacy](../07-security/07-privacy-and-data-protection.md) | public DTOs without phone | E (regex scan) |
| §4.2 | Current vs best score distinguished on result | participant-app | [Shared runtime §8](../05-games/01-shared-game-runtime.md#8-result-screen-shared) | ResultModel | E (AC-011) |
| §6.1 | Normalize Persian/Latin digits; no phone enumeration | identity | [Localization §5](../01-architecture/11-localization-and-rtl.md#5-input-normalization-client-and-server) | OTP request | U, I |
| §7.2 | Card states incl. disabled visible | participant-app | [Lifecycle §1](../02-domain/03-game-session-attempt-lifecycle.md#lobby-card-state-derived-per-participant) | `GET /lobby` | E |
| §8.2, §22.2 | Fair attempt consumption; refresh/background policy | play, runtime | [Lifecycle §4](../02-domain/03-game-session-attempt-lifecycle.md#4-attempt-consumption-policy-answers-spec-82), [Shared runtime §6](../05-games/01-shared-game-runtime.md#6-pause-background-and-interruption-policy) | session timestamps | E, F6, F7 |
| §10.6 | Client never decides reward | rewards | [Fridge Rush §6](../05-games/04-fridge-rush.md#6-mystery-bonus-object-spec-106) | — | code review, RW |
| §11.4 | Layout randomization compatible with validation | game-core | [Vision Hunt §2](../05-games/05-vision-hunt.md#2-scene-generation-deterministic-seeded), ADR-010 | `game_sessions.seed` | GL |
| §12.3 | Deterministic tie-break, no random | scoring | [Best score §2](../02-domain/04-best-score-and-leaderboard.md#2-ranking) | leaderboard index | I |
| §12.4 | No combined leaderboard | — | [Scope](02-scope.md) | — | review |
| §13.4 | Atomic inventory/limits | rewards | [Rewards §4](../02-domain/06-rewards.md#4-evaluation-algorithm-inside-result-transaction-savepoint-protected) | `reward_rules.granted_count`, `reward_codes` | RW-06…08, L5 |
| §13.5 | Result display order; "already received" state | participant-app | [Rewards §7](../02-domain/06-rewards.md#7-presentation-order-client-spec-135) | ResultModel.baseTicket | E, RW-14 |
| §14.2 | Stale/reconnecting indicators on display | admin-app display | [Admin architecture §5](../01-architecture/06-admin-architecture.md#5-public-display-runtime) | SSE heartbeat | F9 |
| §15.3 | No duplicate winners within draw | raffle | [Live raffle §6](../02-domain/07-live-raffle.md#6-duplicate-and-safety-rules) | `draw_winners` UQ | RF-09 |
| §15.4 | Animation separate from selection | display | [Live raffle §5](../02-domain/07-live-raffle.md#5-live-raffle-sequence) | display draw API | RF-15 |
| §16.4 | Hard to create unlimited expensive rewards | rewards + admin | [Rewards §3](../02-domain/06-rewards.md#3-reward-rules), [Operator safety](../06-admin/07-operator-safety-ux.md) | `422 RULE_UNSAFE` | RW-12 |
| §16.5 | Draw results cannot be silently edited | raffle | [Live raffle §4](../02-domain/07-live-raffle.md#4-draw-state-machine) | DB trigger | RF-14 |
| §17.3 | Canonical model carries attempt & best score | integration | [Integration §3](../01-architecture/08-integration-architecture.md#3-canonical-internal-result-model-adapter-input) | `external_deliveries.payload` | contract tests |
| §19.2 | Real-time as hints; recover by re-fetch | realtime | [Real-time events §4](../04-api/07-realtime-events.md#4-client-recovery-algorithm) | snapshots | F3, F13 (AC-022) |
| §20.3 | No scroll conflicts in game canvas | runtime | [PWA §2](../01-architecture/12-pwa-and-mobile.md#2-platform-behaviors) | — | D |
| §21.1 | Realistic anti-cheat | play, ops | [Anti-cheat](../07-security/03-anti-cheat-and-score-integrity.md) | `attempt_flags` | GL (bots), I |
| §23 | No PII to third-party analytics | analytics | [Analytics](../01-architecture/13-analytics-and-telemetry.md) | `analytics_events` | U (redaction), review |
| §25 | Playable muted; sound toggle | runtime | [Shared runtime §10](../05-games/01-shared-game-runtime.md#10-audio-and-haptics) | — | D |
| §27 | Performance budgets | apps, games | [Budgets](../10-performance/01-performance-budgets.md) | — | D, CI size checks, L4 |
| §28 | Accessibility basics | apps, games | [A11y testing](../08-quality/03-accessibility-and-localization-testing.md) | — | E (axe), D |
