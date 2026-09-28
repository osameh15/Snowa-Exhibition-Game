# ADR-008: External Snowa Integration

Status: **Proposed** (contract details pending OQ-04)

## Context
SPEC §17 requires persistence first, non-blocking delivery, reliable retries, idempotency where supported, admin visibility, and no browser access to the external API.

## Options
| Option | Assessment |
|---|---|
| Synchronous call in result request | Violates "do not block result screen"; outage breaks gameplay |
| Fire-and-forget async after commit | Loses deliveries on crash |
| Message broker (RabbitMQ/Kafka) | Extra infrastructure; dual-write problem unless also using an outbox |
| **Transactional outbox table + worker** | Atomic with result; durable; simple; observable via SQL |

## Decision
Transactional outbox (`external_deliveries`) written in the result transaction; worker claims with `FOR UPDATE SKIP LOCKED` and leases; adapter isolates mapping/transport/classification; exponential backoff for 24 h then FAILED for operators; circuit breaker; per-(participant, game) ordering; optional supersede in best-score mode.

## Consequences
+ No lost deliveries; no gameplay dependency on Snowa.
+ Contract changes are adapter-only.
− At-least-once delivery: without receiver idempotency, ambiguous timeouts can cause duplicates (R-05) — mitigated by asking Snowa for an idempotency key.
