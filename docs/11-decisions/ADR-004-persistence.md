# ADR-004: Persistence

Status: **Accepted**

## Context
Strong invariants (uniqueness, row locking, atomic conditional updates, immutable records), relational reporting, moderate scale, single-VPS v1 deployment.

## Options
| Option | Assessment |
|---|---|
| **PostgreSQL 16+** | Transactions, partial unique indexes, `FOR UPDATE SKIP LOCKED` (outbox, code pools, jobs), `LISTEN/NOTIFY` (future multi-instance realtime), JSONB for versioned config, trigram search |
| MySQL 8 | Viable, but lacks partial indexes (base-ticket constraint) and NOTIFY |
| MongoDB | Weaker multi-document invariants for this domain |
| PostgreSQL + Redis from day one | Extra stateful component with no measured need |

## Decision
PostgreSQL 16+ is the **only required durable/stateful service** for v1, running on the same VPS as the application. Outbox, job claiming, rate-limit counters, leaderboard ranks and sessions all live in PostgreSQL. Redis is deferred until measurements prove a need (scaling stage 6).

A separate PostgreSQL server or managed service is an **optional later improvement** (scaling stage 3), not a prerequisite for staging or the exhibition.

Durability requirements regardless of hosting: `fsync`/`synchronous_commit` on (defaults), encrypted off-server backups (daily minimum; more frequent or WAL archiving for event production — see [Backup & recovery](../09-operations/06-backup-recovery-and-reconciliation.md)), restore drill before the event.

## Consequences
+ One system to back up, monitor and restore; invariants enforced in one place.
− Same-host DB shares CPU/RAM with the app (sized and tuned accordingly; measured in load tests).
− The VPS is a single point of failure → mitigated by fast restore, off-server backups and a documented rebuild procedure; accepted for a temporary exhibition.
