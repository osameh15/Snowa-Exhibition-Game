# Score Validation Framework

Source: SPEC §21, §21.1, §11.4. Decision: [ADR-010](../11-decisions/ADR-010-score-validation.md). Security context: [Anti-cheat](../07-security/03-anti-cheat-and-score-integrity.md).

## 1. Principle

The server never trusts `claimedScore`. It **recomputes** the score by replaying the submitted action log through the same deterministic `game-core` code the client used, with the session's seed and pinned config version. Then it applies hard bounds (reject) and statistical plausibility checks (flag).

## 2. Validation layers

| # | Layer | Examples | Failure → |
|---|---|---|---|
| L1 | Transport & schema | JSON shape, sizes, integer types, `actions` ≤ `bounds.max_actions` | `400` (malformed; client bug) |
| L2 | Session binding | session belongs to participant, state STARTED, not past `late_deadline_at`, `configVersion`/`configChecksum` match pinned version, runtime ≥ minimum | reject `SESSION_MISMATCH` / `CONFIG_MISMATCH` / expired |
| L3 | Timing | `server_elapsed ≥ activeMs + pausedMs − tolerance`; `activeMs` consistent with `endReason`; `pausedMs ≤ pauseBudgetMs`; action times monotonic within `[0, activeMs]`; no action inside a pause interval | `REJECTED: TIMING_IMPOSSIBLE` |
| L4 | Replay | `game-core.replay(params, seed, actions)` → `{score, stats}`; every action must be legal in the simulated state (e.g., item exists, round index valid) | illegal action → `REJECTED: INVALID_ACTION` |
| L5 | Consistency | `recomputed.score == claimedScore` | mismatch → `REJECTED: SCORE_MISMATCH` (client and server run identical code, so honest clients never mismatch) |
| L6 | Hard bounds | `score ≤ maxScore(params, seed)` (seed-specific perfect-play ceiling); per-game physical limits (min action intervals) | `REJECTED: OUT_OF_BOUNDS` |
| L7 | Plausibility | per-game statistical thresholds (reaction time distribution, perfect rate, input rate, timing jitter) | `ACCEPTED_FLAGGED` + flag codes |

Server stores claimed score, recomputed score, stats, flags and the raw payload for every attempt (accepted or rejected) [SPEC §12.2].

## 3. Reject vs flag policy

- **Reject** only when a result is impossible for an honest, unmodified client (L2–L6). False rejections of legitimate players must be ~0; any rejection in QA is treated as a bug.
- **Flag** when a result is possible but statistically unusual (L7). Flagged attempts count normally (score, ticket) so legitimate outliers are not punished live; they appear in Admin → Attempts → Flagged for review. Admin may invalidate (audited).
- Before rank-based prizes are awarded or rank-range draws are executed, operators SHOULD review flagged attempts among the relevant top ranks (runbook step).

## 4. Bounds computation

For each config version, `game-core` exposes:
- `maxScore(params, seed)`: simulate optimal play (e.g., Perfect at every opportunity, instant correct placements subject to game rules) → exact seed-specific ceiling.
- `bounds.max_actions`, `bounds.min_action_interval_ms` (physical/logic limits).
- `bounds.plausibility`: thresholds set from prototype calibration (below).

Bounds are stored in `game_config_versions.bounds` at publish time and reviewed by engineering + product.

## 5. Calibration process (Phase 1/3/4 prototypes)

1. Instrument prototypes to collect action logs from internal testers across the device matrix.
2. Compute distributions: score, reaction/timing error, input rates, perfect rate.
3. Set plausibility thresholds at ≈ p99.5 of human data + margin; set hard physical limits well beyond human extremes.
4. Record thresholds + rationale in the config version changelog.
5. Re-run with bots (scripted perfect play) to confirm flags trigger and ceilings hold.

## 6. Duplicate, expiry and retry behavior (all games)

| Case | Behavior |
|---|---|
| Same payload resubmitted | Stored ResultModel returned (`200`) |
| Different payload for a submitted session | `409 RESULT_ALREADY_SUBMITTED`; attempt gets flag `DUPLICATE_MISMATCH`; original result stands |
| Submission after `deadline_at`, before `late_deadline_at` | Validated normally; flag `LATE_SUBMISSION` |
| Submission after `late_deadline_at` | `409 SESSION_EXPIRED`; attempt ABANDONED; payload stored for audit if received |
| Session never started | `409 SESSION_NOT_STARTED` |
| Transient server error | `503 RETRYABLE`; client retries same payload |

## 7. Reason and flag codes

Reject: `SESSION_MISMATCH`, `CONFIG_MISMATCH`, `TIMING_IMPOSSIBLE`, `INVALID_ACTION`, `SCORE_MISMATCH`, `OUT_OF_BOUNDS`, `PAYLOAD_TOO_LARGE`.
Flag: `LATE_SUBMISSION`, `REACTION_TOO_FAST`, `INPUT_RATE_HIGH`, `PERFECT_RATE_HIGH`, `TIMING_JITTER_LOW`, `NEAR_SCORE_CEILING` (≥ 95 % of seed ceiling), `DURING_EMERGENCY_STOP`, `DUPLICATE_MISMATCH`, `REWARD_EVAL_DEFERRED`.

## 8. Per-game summary

| Game | Action log | Key hard checks | Key plausibility flags |
|---|---|---|---|
| Spin Perfect | `[t,"tap"]` | tap cooldown respected by replay; ≤ max taps; score ≤ ceiling | perfect rate, timing-error std-dev |
| Fridge Rush | `[t,"grab",item]`, `[t,"place",item,zone]` | item present at t; grab→place ≥ min drag; single pointer | placement rate, speed-bonus rate |
| Vision Hunt | `[t,"tap",round,objectId,x,y]` | round active; tap within hitbox at t; not in lock window; reaction ≥ physiological min | reaction distribution (median, fast share) |

Details in each game spec.
