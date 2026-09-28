# Raffle and Reward Test Plan

## 1. Raffle

| ID | Test | Expected |
|---|---|---|
| RF-01 | Each population filter on fixture data (all verified, played game X, played any, completed all) | counts match fixture expectations |
| RF-02 | Score threshold GT vs GTE boundary | boundary participant included only for GTE |
| RF-03 | Rank range 1–N with ties at boundary | deterministic tie-break decides membership |
| RF-04 | Ticket min count; VOID tickets excluded | correct |
| RF-05 | Combined filters (AND) | intersection |
| RF-06 | Exclude previous winners on/off; voided draws' winners not excluded | correct |
| RF-07 | Blocked/erased/ranking-excluded never eligible | correct |
| RF-08 | Winner count > eligible | `422 NOT_ENOUGH_ELIGIBLE` (or partial when `allowFewer`) |
| RF-09 | No duplicate winners within draw (many tickets per participant) | unique |
| RF-10 | Reproducibility: `verify-draw` recomputes identical winners | match |
| RF-11 | Statistical fairness: 100k simulated draws, weights {1,2,3} → win frequencies within 99 % CI of expected | pass (chi-square) |
| RF-12 | Uniform weighting fairness | pass |
| RF-13 | Idempotent execute under parallel requests | single COMPLETED |
| RF-14 | Immutability: attempt to update completed draw fields | DB trigger rejects |
| RF-15 | Display API never returns unrevealed winners | verified at each reveal step |
| RF-16 | Reconnect mid-presentation | correct state |
| RF-17 | Audit rows for create/preview/execute_requested/executed/present/reveal/void | present, hash chain valid |

## 2. Rewards

| ID | Test | Expected |
|---|---|---|
| RW-01 | Base ticket once per game; replays show "already active" | 1 ticket |
| RW-02 | Base ticket cannot be disabled via reward APIs | 403/422 |
| RW-03 | Each trigger type (attempt accepted, first completion, new best, all games completed) | fires exactly when defined |
| RW-04 | Score thresholds, game filters, time window boundaries (Tehran time) | correct |
| RW-05 | Probability: 10 000 evaluations at 25 % → 2 500 ± 3σ | pass |
| RW-06 | Total limit under concurrency | exact |
| RW-07 | Per-participant limit across games under concurrency | exact |
| RW-08 | Code pool allocation uniqueness, pool exhaustion → no grant, result still accepted | pass |
| RW-09 | Exclusive group grants at most one | pass |
| RW-10 | Extra tickets rows created and counted in draws | pass |
| RW-11 | Reward evaluation exception → attempt accepted, `REWARD_EVAL_DEFERRED`, later re-evaluation grants once | pass |
| RW-12 | Unsafe rule creation rejected (physical prize without limit, unlimited without Super Admin) | 422/403 |
| RW-13 | Revoke grant; code VOID; inventory return option | pass |
| RW-14 | Result screen order and states [SPEC §13.5] | E2E pass |
