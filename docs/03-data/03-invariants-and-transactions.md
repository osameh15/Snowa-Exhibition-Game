# Invariants, Transactions and Idempotency

Each invariant lists **how** it is enforced. "DB" = database constraint (last line of defense), "TX" = transactional logic under lock, "JOB" = reconciliation detection.

## 1. Invariants

| ID | Invariant | Enforcement |
|---|---|---|
| INV-01 | One participant per phone | DB: UQ `participants.phone_e164`, `phone_hash`; TX: `INSERT … ON CONFLICT DO NOTHING` then select |
| INV-02 | ≤ 1 open session per participant + game | DB: partial UQ on `game_sessions (participant_id, game_id) WHERE state IN ('ISSUED','STARTED')`; TX: progress row lock |
| INV-03 | Attempts never exceed allowance at start | TX: `SELECT … FOR UPDATE` progress; check `attempts_used < attempt_limit + bonus_attempts` using `game_settings` read in same tx; increment |
| INV-04 | Attempt numbers unique & contiguous | DB: UQ `(event, participant, game, attempt_number)`; TX: number = `attempts_used + 1` under lock |
| INV-05 | One attempt per session; one result per attempt | DB: UQ `attempts.session_id`; TX: session state `STARTED → SUBMITTED` transition under lock; `attempt_payloads` PK = attempt_id |
| INV-06 | Best score = max valid attempt score | TX: updated on every attempt status change (accept, invalidate, restore) under progress lock; JOB: `integrity.reconcile` compares with `MAX()` |
| INV-07 | ≤ 1 BASE ticket per participant + game + event | DB: partial UQ; TX: progress lock; JOB: reconcile |
| INV-08 | Single-use code → ≤ 1 grant | DB: UQ `reward_codes.grant_id`, `reward_grants.code_id`; TX: `FOR UPDATE SKIP LOCKED` allocation |
| INV-09 | Grants ≤ total_limit and ≤ per-participant limit | TX: conditional `UPDATE … WHERE granted_count < total_limit`, conditional counter upsert |
| INV-10 | Completed draw immutable | DB: trigger + app role privileges; TX: state check under row lock |
| INV-11 | Participant wins ≤ once per draw | DB: UQ `(draw_id, participant_id)` on `draw_winners`; algorithm without replacement |
| INV-12 | Every valid attempt has a delivery record | TX: outbox insert in result tx; DB: UQ `(attempt_id, kind)`; JOB: reconcile valid attempts without delivery |
| INV-13 | Scores are non-negative integers within bounds | DB: CK ≥ 0; TX: validator bounds from config version |
| INV-14 | Rank order is total and deterministic | Index + tie-break keys; `best_attempt_seq` UQ |
| INV-15 | Audit row exists for every admin mutation | TX: audit insert in same tx (service layer wrapper `withAudit`) |
| INV-16 | PUBLISHED config versions immutable | DB trigger |
| INV-17 | At most one LIVE event | DB: partial UQ |

## 2. Transaction specifications

Isolation: `READ COMMITTED` + explicit row locks (predictable, low-abort). All multi-row invariants are protected by locking a single "owner" row (`participant_game_progress`, `reward_rules`, `draws`), not by SERIALIZABLE.

### T-START (start session)
```text
BEGIN
  SELECT * FROM game_sessions WHERE id=$s AND participant_id=$p FOR UPDATE
  if state = STARTED → return existing (idempotent)
  if state != ISSUED or now > start_by → error
  SELECT * FROM game_settings WHERE (event, game) — must be ENABLED; event LIVE
  SELECT * FROM participant_game_progress WHERE (event,p,game) FOR UPDATE   -- created at session issue if absent
  if attempts_used >= attempt_limit + bonus_attempts → CANCELLED, error ATTEMPTS_EXHAUSTED
  UPDATE progress SET attempts_used = attempts_used + 1
  INSERT attempts(status IN_PROGRESS, attempt_number = attempts_used)
  UPDATE game_sessions SET state=STARTED, started_at=now(), deadline_at=…, late_deadline_at=…
COMMIT
```

### T-SUBMIT (result)
Documented in [Backend architecture §4](../01-architecture/05-backend-architecture.md#4-result-acceptance-pipeline). Lock order: `game_sessions` row → `participant_game_progress` row → `reward_rules` rows (ascending id) → `reward_codes` (SKIP LOCKED). Same order everywhere prevents deadlocks.

### T-INVALIDATE (admin invalidates attempt)
```text
BEGIN
  lock progress row; lock attempt
  attempt.status_before_invalidation = status; status = INVALIDATED
  recompute best: SELECT … FROM attempts WHERE valid ORDER BY attempt_score DESC, accepted_at ASC, seq ASC LIMIT 1
  update progress best_* (or NULL)
  apply ticket policy (OQ-23)
  audit; enqueue CORRECTION delivery if contract supports (OQ-04)
COMMIT → publish best_score_changed / leaderboard_changed on RealtimeBus
```

### T-DRAW-EXECUTE
See [Live raffle §5](../02-domain/07-live-raffle.md#5-live-raffle-sequence).

## 3. Idempotency keys

| Operation | Key | Stored in | Duplicate behavior |
|---|---|---|---|
| OTP request | `Idempotency-Key` header (client UUID) | `idempotency_keys` (scope anon/ip) | Same response; no second SMS |
| OTP verify | challenge id (state machine) | `otp_challenges` | VERIFIED challenge cannot verify again → `409 CHALLENGE_USED`; client with cookie proceeds |
| Create session | `Idempotency-Key` + INV-02 | `game_sessions.create_idempotency_key` | Returns open ISSUED session |
| Start session | session id + state | `game_sessions` | STARTED → same response |
| Submit result | **session id** (natural key) + payload SHA-256 | `attempt_payloads.payload_sha256` | Same hash → stored result; different → `409 RESULT_ALREADY_SUBMITTED`, flag `DUPLICATE_MISMATCH` |
| Base ticket | `(event, participant, game)` | partial UQ | no-op |
| Reward grant | `(rule_id, source_attempt_id)` | UQ | no-op |
| Extra tickets | `(reward_grant_id, ordinal)` | UQ | no-op |
| Draw execute | `Idempotency-Key` + draw state | `draws.execute_idempotency_key` | Returns completed draw |
| Admin mutations | `Idempotency-Key` | `idempotency_keys` (scope admin) | Stored response |
| External delivery | `external_deliveries.id` | outbox | Sent as idempotency header if Snowa supports (OQ-04) |
| Analytics batch | `batch_id` | `analytics_events` UQ `(anon_id, client_event_id)` | Dropped duplicates |

`idempotency_keys` protocol: insert `(scope,key,request_hash,IN_PROGRESS)`; on conflict: if `COMPLETED` and same hash → replay stored response; if hash differs → `422 IDEMPOTENCY_KEY_REUSED`; if `IN_PROGRESS` → `409 REQUEST_IN_PROGRESS` (client retries later).

## 4. Concurrency scenarios

| Scenario | Outcome |
|---|---|
| Same participant opens game in two tabs and taps start in both | Both `POST /game-sessions` return the same ISSUED session (INV-02); only one `/start` creates an attempt; the other tab gets STARTED state and shows "playing in another tab" |
| Two submissions of the same session race | Session row lock serializes; second sees SUBMITTED |
| Two participants win the last unit of a reward simultaneously | Conditional update: exactly one gets `granted_count < total_limit` true |
| Admin lowers attempt limit while participant is in pre-game | `/start` re-checks inside tx → `ATTEMPTS_EXHAUSTED`, session CANCELLED, no attempt consumed |
| Admin disables game during gameplay | STARTED session submits normally |
| Draw executed while submissions stream in | Snapshot reflects committed state at execution tx time (READ COMMITTED statement snapshot of the `INSERT … SELECT`) |
