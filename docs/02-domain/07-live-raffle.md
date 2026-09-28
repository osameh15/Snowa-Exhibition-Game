# Live Raffle (Draw) Model

Source: SPEC §15, §16.5, PD-10, AC-015, AC-016, §22.1 ("never show a draw winner until persisted"). Decision: [ADR-011](../11-decisions/ADR-011-raffle-selection.md). API: [Raffle API](../04-api/06-raffle-api.md).

## 1. Ticket vs draw

| | Raffle ticket | Draw |
|---|---|---|
| What | Entitlement unit (chance) | One execution selecting winners |
| Created by | Result pipeline / reward engine / admin | Admin command |
| Changes over time | ACTIVE ↔ VOID | Immutable once COMPLETED (except reveal progress and VOID marking) |
| Used by | Draw eligibility & weighting | Presentation, audit, prize fulfillment |

## 2. Eligibility criteria

Criteria are a JSON document validated by schema, combined with **AND** [SPEC §15.1]:

| Criterion | Schema | Semantics (evaluated at execution time) |
|---|---|---|
| `population` | `ALL_VERIFIED` \| `PLAYED_ANY` \| `PLAYED_GAME{game}` \| `COMPLETED_ALL` | Verified = participant ACTIVE with name. Played = ≥1 valid attempt. Completed all = valid attempt in all three games |
| `score` | `{game, op: GT\|GTE, value}` | on best score of that game |
| `rank_range` | `{game, from, to}` | rank (per [ranking rules](04-best-score-and-leaderboard.md#2-ranking)) at execution time, inclusive |
| `tickets` | `{min_count ≥ 1}` | active tickets (all sources) |
| `exclude_previous_winners` | bool (default `true`, OQ-10) | excludes winners of COMPLETED, non-VOID draws of this event |
| always | — | BLOCKED/ERASED participants and ranking-excluded participants are never eligible |

Weighting (OQ-08 — open product decision):

| `weighting` | Entry weight | Note |
|---|---|---|
| `TICKETS` (**recommended default**) | number of ACTIVE tickets | Consistent with SPEC §13.3 "extra ticket adds extra entries"; participants with 0 tickets are ineligible |
| `UNIFORM` | 1 per eligible participant | For "top-N ranking" draws where tickets are irrelevant |

In both modes a participant wins **at most once per draw** (selection without replacement at participant level) [SPEC §15.3].

## 3. Selection algorithm (v1)

```text
inputs: entries[] = eligible participants ordered by participant_id ASC, each with weight w_i > 0
        k = winner_count (1 ≤ k ≤ |entries|)
        seed = 32 bytes from CSPRNG, generated inside the execution transaction after the snapshot is built
prng(counter) = HMAC-SHA256(key = seed, msg = draw_id ‖ ":" ‖ counter)  → 256-bit integer
uniform(n): take 64-bit chunks from prng output, rejection-sample to avoid modulo bias, counter++ as needed
repeat k times:
    W = sum of remaining weights
    r = uniform(W)
    walk remaining entries in order accumulating weights; choose first entry with cumulative > r
    append to winners (position = 1..k); remove it from remaining
algorithm_version = "weighted-wor-hmac-sha256-v1"
```

Properties: unbiased, reproducible from `(entries, seed, draw_id, algorithm_version)`, and not influenced by the browser. Complexity O(n·k) — at 50k entries × 100 winners = 5M steps (< 1 s).

## 4. Draw state machine

```mermaid
stateDiagram-v2
  [*] --> DRAFT: create (criteria, winner_count, weighting)
  DRAFT --> DRAFT: edit / preview count
  DRAFT --> CANCELLED: cancel
  DRAFT --> COMPLETED: execute (single tx: snapshot + seed + winners)
  COMPLETED --> PRESENTING: start presentation on displays
  PRESENTING --> PRESENTING: reveal next winner (revealed_count++)
  PRESENTING --> REVEALED: all winners revealed
  COMPLETED --> REVEALED: reveal all at once
  COMPLETED --> VOIDED: Super Admin void (reason)
  PRESENTING --> VOIDED
  REVEALED --> VOIDED
  CANCELLED --> [*]
  REVEALED --> [*]
  VOIDED --> [*]
```

- There is no "re-roll". A contested draw is `VOIDED` (record kept, reason required) and a **new** draw is created with `replaces_draw_id`. Both appear in history.
- Snapshot, seed, winners, criteria, `eligible_count`, `total_weight` are write-once (DB trigger rejects updates after COMPLETED).

## 5. Live raffle sequence

```mermaid
sequenceDiagram
  autonumber
  actor AD as Admin
  participant UI as Admin console
  participant API as API (raffle)
  participant DB as PostgreSQL
  participant DSP as Public display(s)
  AD->>UI: build criteria, winners = 5
  UI->>API: POST /admin/draws (DRAFT)
  UI->>API: POST /admin/draws/:id/preview
  API->>DB: eligibility query (read-only)
  API-->>UI: {eligibleCount, totalWeight, asOf}
  AD->>UI: confirm (typed confirmation: winner count + event name)
  UI->>API: POST /admin/draws/:id/execute (Idempotency-Key, version)
  API->>DB: TX1: audit draw.execute_requested (committed)
  API->>DB: TX2 BEGIN · SELECT draw FOR UPDATE · state must be DRAFT
  API->>DB: INSERT draw_entries SELECT … (eligibility query, ordered)
  API->>API: seed = CSPRNG(32) · snapshot_hash = SHA-256(canonical entries)
  API->>API: select k winners (algorithm v1)
  API->>DB: INSERT draw_winners · UPDATE draw → COMPLETED (seed, hash, counts, executed_by/at, criteria snapshot, config snapshot)
  API->>DB: audit draw.executed · analytics raffle_completed · COMMIT
  API-->>UI: {state: COMPLETED, winners (admin view, full data by permission)}
  AD->>UI: "present on display"
  UI->>API: POST /admin/draws/:id/present
  API->>DB: state PRESENTING, revealed_count = 0
  API-->>DSP: SSE draw_presentation_started {drawId, eligibleCount, winnerCount}
  DSP->>DSP: play suspense animation (cosmetic only)
  AD->>UI: "reveal next"
  UI->>API: POST /admin/draws/:id/reveal-next
  API->>DB: revealed_count++ (persisted)
  API-->>DSP: SSE draw_winner_revealed {drawId, position}
  DSP->>API: GET /display/v1/draws/:id (returns only positions ≤ revealed_count, public names)
  DSP->>DSP: land animation on returned winner
```

Displays never receive unrevealed winners, so inspecting the display browser cannot leak results early; the animation only visualizes an already-persisted result [SPEC §15.4].

## 6. Duplicate and safety rules

| Risk | Protection |
|---|---|
| Admin double-clicks execute | Idempotency-Key + row lock; second call returns the same COMPLETED draw |
| Two admins execute the same draft | Row lock + state check; one wins, other gets the completed result |
| Admin executes twice to fish for a preferred winner | Each execution needs a new draft; all executions + requests audited; voiding requires Super Admin + reason; history visible |
| Winner count > eligible | `422 NOT_ENOUGH_ELIGIBLE` (no partial draw unless `allow_fewer = true`) |
| Display disconnect mid-reveal | Persisted `revealed_count`; reconnect renders current state |
| Participant with many tickets wins multiple times | Selection without replacement at participant level |

## 7. Audit & verification

Stored per draw: `id`, `event_id`, `criteria` (JSON), `weighting`, `winner_count`, `eligible_count`, `total_weight`, `preview_count` (last preview), `snapshot_hash`, `seed`, `algorithm_version`, `app_version`, `executed_by`, `executed_at`, `config_snapshot` (event policies relevant to draws), `replaces_draw_id`, `void_reason`, `voided_by/at`.

A CLI `verify-draw <id>` (Phase 5) recomputes the snapshot hash from `draw_entries` and the winners from the seed, and must reproduce `draw_winners` exactly. The export for external auditors includes entries with pseudonymous ids.
