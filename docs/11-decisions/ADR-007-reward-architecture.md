# ADR-007: Reward Architecture

Status: **Proposed**

## Context
Base ticket is fixed, mandatory and once per game [SPEC §13.1–13.2]; extra rewards are admin-configured with probability, limits, windows and inventory [SPEC §13.3–13.4]; games must not contain reward logic.

## Options
| Option | Assessment |
|---|---|
| A. Everything as reward rules (base ticket = locked system rule) | Uniform engine, but the most important invariant would depend on rule configuration and could be misconfigured |
| **B. Base ticket as a dedicated pipeline step + engine for extra rewards** | Invariant enforced by code + unique index; engine flexibility where needed |
| C. Rewards evaluated asynchronously after result | Simpler result tx, but result screen couldn't show rewards immediately [SPEC §13.5] |

## Decision
Option B. The reward engine evaluates `Definition + Rule + Inventory → Grant` synchronously inside the result transaction under a savepoint, with atomic conditional updates for inventory and per-participant limits and `SKIP LOCKED` code allocation. Failure of the engine never blocks accepting the attempt (deferred re-evaluation). Extensible via `type` + fulfillment handler registry.

## Consequences
+ Base ticket cannot be broken by admin configuration.
+ Rewards appear on the result screen immediately; exactness under concurrency.
− Hot reward rules serialize their grants (row lock) — acceptable at event scale; many rules per result increase tx time (keep active rules ≤ ~20).
