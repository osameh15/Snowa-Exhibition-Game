# ADR-010: Score Validation by Deterministic Replay

Status: **Accepted** — non-negotiable; unaffected by infrastructure simplification.

## Context
The browser is untrusted [SPEC §21]; `score = 999999999` must not be accepted; SPEC §11.4 asks how layout randomization interacts with validation; gameplay must not depend on network round trips [SPEC §27].

## Options
| Option | Assessment |
|---|---|
| Trust claimed score with max bound | Trivially exploitable up to the bound |
| Heuristic checks on summary stats only | Stats can be faked consistently |
| Server-driven gameplay (per-round/per-input round trips) | Breaks offline tolerance; adds latency on venue networks |
| Client-side secrets / payload encryption / obfuscation as trust anchor | **Rejected**: client code is observable; any key shipped to the browser is known to attackers |
| **Seeded deterministic simulation + action-log replay on server** | Strong against direct tampering; no network during play; supports randomized layouts |

## Decision
- Each attempt uses a **server-issued game session** carrying: session id, participant, game, attempt number (assigned at start), server seed, pinned game config version + checksum, issued/start timestamps, start-by/deadline/late-deadline, status.
- Gameplay is a deterministic function of `(configuration, server seed, recorded player inputs)` implemented once in `packages/game-core` with integer arithmetic, used by both the Phaser client and the server.
- The client submits the timestamped input/action log. The server replays it and computes the **authoritative score**; the claimed score is only compared (mismatch → reject), never trusted.
- Required checks: schema and size; session ownership/state/expiry; config-version pinning; monotonic timestamps within duration; pause accounting; server wall-clock lower bound on elapsed time; game duration bounds; impossible input-rate detection; legal-action replay; seed-specific maximum score; duplicate-submission and replay protection (one result per session, payload hash); suspicious-but-possible → `ACCEPTED_FLAGGED`; impossible → `REJECTED`; expired session → rejected.
- Cross-engine golden tests (Node/V8 vs Chromium vs WebKit) are mandatory CI gates.

## Consequences
+ Arbitrary score submission impossible; duplicates/replays harmless; layouts vary per session yet remain verifiable.
− Bots that generate plausible logs remain possible (accepted residual risk; detection via flags and review).
− Strict determinism discipline required.
− Seed known client-side → Vision Hunt positions computable by modified clients (mitigated by reaction-time plausibility and hitbox checks).
