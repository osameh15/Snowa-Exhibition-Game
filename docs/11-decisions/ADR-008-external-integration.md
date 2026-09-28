# ADR-008: External Snowa Integration

Status: **Accepted** (outbox model); external contract details pending OQ-04

## Context
SPEC §17 requires persistence first, non-blocking delivery, reliable retries, idempotency where supported, admin visibility, and no browser access to the external API. v1 has no message broker and no separate worker process.

## Options
| Option | Assessment |
|---|---|
| Synchronous call in result request | Violates "do not block result screen" |
| Fire-and-forget async after commit | Loses deliveries on crash |
| Message broker (RabbitMQ/Kafka) or Redis queue | Extra infrastructure; still needs an outbox to avoid dual writes |
| **PostgreSQL transactional outbox + background sender** | Atomic with result; durable; observable via SQL; no new infrastructure |

## Decision
Transactional outbox (`external_deliveries`) written in the result transaction. A background sender in the `jobs` module (in-process in v1, separable later) claims due rows with `FOR UPDATE SKIP LOCKED` and time-limited leases; the `ExternalResultAdapter` port (`SnowaResultAdapter` implementation) isolates mapping, transport and response classification; exponential backoff for 24 h then `FAILED` for operators; circuit breaker; per-(participant, game) ordering; optional supersede in best-score mode. No Kafka, RabbitMQ or Redis queues in v1.

## Consequences
+ No lost deliveries across restarts (lease recovery); no gameplay dependency on Snowa.
+ Contract changes and future brands are adapter-only.
− At-least-once delivery: without receiver idempotency, ambiguous timeouts can duplicate (R-05).
