# ADR-004: Persistence

Status: **Proposed** (OQ-02 affects hosting of the database)

## Context
Strong invariants (uniqueness, row locking, atomic conditional updates, immutable records), relational reporting, moderate scale, small ops team.

## Options
| Option | Assessment |
|---|---|
| **PostgreSQL 16+** | Transactions, partial unique indexes, `FOR UPDATE SKIP LOCKED` (outbox, code pools), `LISTEN/NOTIFY`, JSONB for versioned config, trigram search, mature managed offerings and self-hosting |
| MySQL 8 | Viable (SKIP LOCKED available) but lacks partial indexes (base-ticket constraint needs workaround) and NOTIFY |
| MongoDB | Weaker multi-document invariants for this domain; reporting harder |
| PostgreSQL + Redis from day one | Redis helps at very high scale (rate limits, ZSET leaderboards, pub/sub) but adds a stateful component to operate and fail |

## Decision
PostgreSQL as the **only** stateful dependency in v1. Rate limits, leaderboard ranks, pub/sub and outbox all use PostgreSQL. Redis is a documented upgrade path triggered by load-test evidence ([Real-time & leaderboard performance §3](../10-performance/03-realtime-and-leaderboard-performance.md#3-upgrade-paths-only-if-load-tests-fail)).

Managed PostgreSQL with PITR is preferred; otherwise self-hosted primary + streaming replica + WAL archiving.

## Consequences
+ One system to back up, monitor and fail over; invariants enforced in one place.
− Rate-limit counters add small write load to the DB (acceptable at A-01 scale).
− Rank queries are O(rank); acceptable ≤ ~100k ranked rows per game.
