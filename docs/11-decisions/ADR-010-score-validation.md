# ADR-010: Score Validation by Deterministic Replay

Status: **Proposed**

## Context
The browser is untrusted [SPEC §21]; `score = 999999999` must not be accepted; SPEC §11.4 asks how layout randomization interacts with validation; gameplay must not depend on network round trips [SPEC §27].

## Options
| Option | Assessment |
|---|---|
| Trust claimed score with max bound | Trivially exploitable up to the bound |
| Heuristic checks on summary stats only | Weak; stats can be faked consistently |
| Server-driven gameplay (server sends each round / validates each input online) | Strong, but breaks offline tolerance and adds latency to gameplay on congested venue networks |
| **Seeded deterministic simulation + action-log replay on server** | Strong against direct tampering; no network during play; supports randomized layouts via server seed |

## Decision
Each session gets a server seed and a pinned config version. Games are deterministic functions of `(params, seed, inputs)` implemented in `packages/game-core` with integer arithmetic. Clients submit the timestamped input log; the server replays it, computes the authoritative score, applies hard bounds and plausibility flags. Claimed score must match (mismatch → reject), which also detects client/server drift early.

## Consequences
+ Arbitrary score submission impossible; replays/duplicates harmless; layouts vary per session yet remain verifiable.
+ Golden-log tests pin scoring behavior across releases.
− Bots that generate plausible logs remain possible (accepted residual risk; detection via flags).
− Requires strict determinism discipline and cross-engine tests (V8 vs JavaScriptCore).
− Seed known client-side → Vision Hunt positions computable by modified clients (mitigated by reaction plausibility).
