# ADR-011: Raffle Selection Algorithm and Snapshot

Status: **Proposed** (weighting semantics pending product decision OQ-08; cross-draw winner policy OQ-10)

## Context
SPEC §15 requires server-side selection, eligibility snapshot, duplicate prevention within a draw, auditability, separation of animation from selection.

## Options
| Topic | Options | Choice |
|---|---|---|
| Randomness | `Math.random` / CSPRNG per pick / CSPRNG seed + deterministic stream | **CSPRNG seed + HMAC-SHA256 stream** → unpredictable and reproducible |
| Snapshot | store criteria only / store entries | **Store entries** (criteria re-evaluation later would differ as data changes) |
| Weighting | uniform / ticket-weighted | **Configurable per draw, default ticket-weighted** (recommended reading of SPEC §13.3; needs confirmation) |
| Duplicates | with replacement / without replacement at participant level | **Without replacement** [SPEC §15.3] |
| Re-runs | allow re-roll / void + new draw | **Void + new draw** with full history |
| Public verifiability | none / hash commitment | Snapshot hash displayed optionally; seed + pseudonymized entries exportable |

## Decision
Algorithm `weighted-wor-hmac-sha256-v1` as specified in [Live raffle §3](../02-domain/07-live-raffle.md#3-selection-algorithm-v1), executed atomically with snapshot persistence; immutable thereafter.

## Consequences
+ Any draw can be independently recomputed; operators cannot influence outcome without leaving audit traces.
− Snapshot storage O(eligible) per draw (fine: 50k rows ≈ few MB).
− Weighting choice must be approved and communicated in raffle terms.
