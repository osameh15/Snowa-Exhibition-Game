# Backend Architecture

Technology recommendation and alternatives: [ADR-002](../11-decisions/ADR-002-backend-technology.md). This document is written so that the *behavior* holds regardless of the final framework; examples assume Node.js + TypeScript + Fastify + PostgreSQL.

## 1. Process model

| Process | Entrypoint | Runs | Count |
|---|---|---|---|
| `api` | `server/api.ts` | HTTP REST, SSE streams, NOTIFY listener | 2+ (stateless) |
| `worker` | `server/worker.ts` | Outbox delivery, session expiry sweeper, reward re-evaluation queue, integrity reconciliation, metrics aggregation | 1 (safe to run 2: all jobs claim rows with `FOR UPDATE SKIP LOCKED`) |
| `migrate` | `server/migrate.ts` | Schema migrations (one-shot, before rollout) | job |

No in-memory state is authoritative. Anything lost on restart is either recomputable (caches, SSE ring buffer) or durable in PostgreSQL.

## 2. Request pipeline

```mermaid
flowchart LR
  REQ["HTTP request"] --> RID["request id + structured log context"]
  RID --> SZ["size limit (body ≤ 64 KB; result ≤ 128 KB)"]
  SZ --> AUTHN["authenticate<br/>(participant cookie / admin cookie / display token)"]
  AUTHN --> RL["rate limit (per phone / session / IP class)"]
  RL --> VAL["schema validation (zod, shared contracts)"]
  VAL --> AUTHZ["authorize (ownership / RBAC permission)"]
  AUTHZ --> IDEM["idempotency check (for mutating endpoints)"]
  IDEM --> H["handler → module service"]
  H --> TX[("single DB transaction per command")]
  TX --> POST["after-commit: publish realtime hints, metrics"]
  POST --> RES["response (typed, no internal details)"]
```

## 3. Transaction boundaries (critical commands)

| Command | One transaction contains | Locks |
|---|---|---|
| Verify OTP | challenge consume, participant upsert, session insert, audit (first login) | challenge row |
| Create game session | settings read, open-session check, session insert | `participant_game_progress` row (`FOR UPDATE`) |
| Start session | attempt-limit check, attempt insert (`attempt_number`), progress `attempts_used++`, session → STARTED | progress row |
| Submit result | attempt validation outcome, payload insert, flags, best-score update, base ticket (unique), reward evaluation (savepoint), outbox insert, analytics insert | progress row; reward rule rows / codes (`SKIP LOCKED`) |
| Execute draw | draw state transition, snapshot insert, winner insert, audit | draw row (`FOR UPDATE`) |
| Admin config change | settings update (optimistic `version` check), audit | settings row |

Realtime publications and non-critical side effects occur **after commit**; if the process dies between commit and publish, clients recover by re-fetch (hints are not authoritative).

## 4. Result-acceptance pipeline

The heart of the system. Executed inside the submit transaction unless noted.

```mermaid
flowchart TD
  A["POST /game-sessions/:id/result"] --> B{"Idempotent replay?<br/>(session already SUBMITTED)"}
  B -- "same payload hash" --> B1["return stored result (200)"]
  B -- "different payload" --> B2["409 RESULT_ALREADY_SUBMITTED + flag DUPLICATE_MISMATCH"]
  B -- no --> C["Lock progress row; load session + config version"]
  C --> D{"Session owned, STARTED,<br/>within deadline or late window?"}
  D -- no --> D1["reject: SESSION_EXPIRED / NOT_STARTED / FORBIDDEN"]
  D -- yes --> E["Structural checks: runtime/config version, log size, monotonic timestamps"]
  E --> F["Replay action log with game-core → recomputed score + stats"]
  F --> G{"recomputed == claimed?<br/>within hard bounds?<br/>timing plausible?"}
  G -- "hard violation" --> G1["attempt REJECTED (reason code)"]
  G -- "ok / soft anomalies" --> H["attempt ACCEPTED or ACCEPTED_FLAGGED;<br/>attempt_score = recomputed"]
  H --> I["Best score: if attempt_score > best → update best_score, best_achieved_at, best_attempt_id/seq"]
  I --> J["First valid completion? → insert BASE ticket<br/>(unique partial index guards duplicates)"]
  J --> K["SAVEPOINT reward evaluation<br/>(rules, probability, atomic inventory)"]
  K -- error --> K1["rollback to savepoint; enqueue re-evaluation job; flag REWARD_EVAL_DEFERRED"]
  K --> L["Insert external_delivery (outbox)"]
  K1 --> L
  G1 --> M
  L --> M["Session → SUBMITTED; commit"]
  M --> N["After commit: NOTIFY attempt_accepted / best_score_changed / leaderboard_changed"]
  N --> O["Response: authoritative ResultModel incl. rank"]
```

Rank in the response is computed after commit with a single indexed query (see [Leaderboard rules](../02-domain/04-best-score-and-leaderboard.md)).

## 5. Configuration and caching

| Item | Source | Cache | Invalidation |
|---|---|---|---|
| `game_settings` | DB | per-process, 2 s TTL | NOTIFY `config_changed` clears immediately |
| `game_config_versions` | DB | per-process, forever | immutable |
| `reward_rules` (active) | DB | per-process, 2 s TTL | NOTIFY `rewards_changed` |
| Leaderboard top-N | DB query | per-process per game, 1 s | time-based; SSE hints throttled to 1/s |
| Participant session | DB | none (single indexed lookup) | — |

Attempt-limit and availability checks at session **creation/start** always read the DB row inside the transaction (never cache-only), so admin changes apply immediately to new sessions [SPEC §16.2].

## 6. Background jobs (worker)

| Job | Trigger | Behavior |
|---|---|---|
| `outbox.deliver` | loop, 1 s idle poll + NOTIFY wake-up | claim `PENDING`/`RETRY_SCHEDULED` rows due now, `SKIP LOCKED`, batch ≤ 20, call adapter, record attempt |
| `sessions.expire` | every 15 s | ISSUED past `start_by` → `EXPIRED` (no attempt); STARTED past `late_deadline_at` → attempt `ABANDONED`, session `EXPIRED` |
| `rewards.reevaluate` | queue rows | re-run reward evaluation for attempts flagged `REWARD_EVAL_DEFERRED` (idempotent via grant unique keys) |
| `integrity.reconcile` | every 10 min + on demand | verify `best_score` equals max of valid attempts; verify base tickets vs first valid completions; report mismatches (no auto-fix without admin) |
| `metrics.aggregate` | every 10 s | dashboard counters (participants, active players, attempts per game) into memory + SSE |
| `retention.purge` | daily (disabled until OQ-13 answered) | apply retention policy |

## 7. Failure behavior summary

| Failure | Behavior |
|---|---|
| API replica crash | Proxy routes to other replica; in-flight requests fail and are retried by client with the same idempotency key |
| DB transient error (serialization/connection) | Retry transaction up to 3× with jitter inside handler; then `503 RETRYABLE` → client retries |
| DB down | `503`; participant sees retry state; pending result stays in `localStorage`; late-submission window covers short outages |
| Worker down | Outbox backlog grows; expiry sweeps pause (sessions are also lazily expired when touched) |
| Clock skew between replicas | All server timestamps come from PostgreSQL `now()`/`clock_timestamp()` inside transactions, not from app hosts |
