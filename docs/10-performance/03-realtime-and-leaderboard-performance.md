# Real-Time and Leaderboard Performance

## 1. Load assumptions (A-01; confirm with OQ-05)

These are assumptions, not measurements; they drive test targets, not server sizing. v1 runs everything (SSE hub, jobs, PostgreSQL) on one VPS; the numbers below are validated on that topology by load tests ([Load testing](../08-quality/04-load-and-resilience-testing.md)).

| Quantity | Design | Test target |
|---|---|---|
| Result submissions | 20/s | 100/s |
| Best-score changes | ≤ 50 % of results | — |
| SSE connections | ≤ 600 | 2 600 |
| Leaderboard top-N reads | ≤ 100/s | 500/s |
| Ranked rows per game | 10k | 50k |

## 2. Server-side cost model

| Operation | Cost | Mitigation |
|---|---|---|
| Result pipeline | 1 tx, ~10–15 statements, lock on one progress row (+ reward rows if rules match) | Keep rules few; index coverage; tx < 30 ms |
| Rank of one participant | O(rank) index-only scan | Acceptable to ~100k; else cached rank buckets or Redis ZSET |
| Top-N page | O(N) index scan | 1 s per-process cache per game |
| Hint fan-out | in-process RealtimeBus → coalesce → SSE write per client (NOTIFY per instance at stage 5) | Coalescing (≤ 1/s/game) bounds writes to `clients × games × 1/s` |
| Client re-fetch storm after hint | N viewers × 1 fetch | `topChanged` flag + jittered fetch (0–500 ms) + 1 s cache |

## 3. Upgrade paths (only if load tests fail)

1. Larger VPS (vertical resize), then separate PostgreSQL and a separate worker — see [scaling path](../01-architecture/10-deployment-topology.md#5-incremental-scaling-path-future-options-not-v1-requirements).
1. More Fastify instances (stateless) with the NOTIFY realtime bus.
2. Read replica for reports/leaderboard reads.
3. Redis: rate limits, top-N cache, pub/sub fan-out, ZSET per game for rank (score encoded with tie-break: `score * 2^32 + (MAX_TS − achieved_seq)`).
4. Pre-computed rank snapshots every 1 s for deep ranks.

## 4. Client-side

- Display and admin apply diffs, not full re-renders; virtualized long tables (admin full leaderboard).
- Participant leaderboard screen closes SSE when hidden; re-opens and re-fetches on visible.
