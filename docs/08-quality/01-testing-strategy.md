# Testing Strategy

## 1. Test layers

| Layer | Scope | Tooling (recommended) | Runs |
|---|---|---|---|
| Unit | Pure functions: normalization, formatting, scoring rules, PRNG, reward condition matching, draw selection, retry backoff | Vitest | every commit |
| Game-logic (game-core) | Deterministic simulation, replay, bounds, golden logs | Vitest + fixtures | every commit |
| Cross-engine determinism | Same golden logs produce identical scores in Node (server) and browsers (Chromium, WebKit) | Playwright running game-core in page | every commit touching game-core |
| Integration (backend) | Handlers + real PostgreSQL (Testcontainers): transactions, constraints, idempotency, concurrency | Vitest + Testcontainers | every commit |
| Contract | API responses match `contracts` schemas; Snowa adapter mapping fixtures | zod + fixtures | every commit |
| Component (UI) | Vue components: RTL rendering, states, error mapping | Vitest + Vue Test Utils | every commit |
| E2E | Full journeys in browsers with fake OTP & fake Snowa receiver | Playwright (Chromium, WebKit, mobile emulation) | every merge to main + nightly |
| Real-device | Device matrix, performance, touch, audio, background behavior | Manual + remote device lab if available | per milestone + event rehearsal |
| Load | API + DB + SSE at target profiles | k6 (or equivalent) | per milestone, before event |
| Security | Dependency scan, secret scan, authz tests, basic DAST | CI tools + manual review | CI + pre-event |
| Failure injection | Dependency outages, restarts, latency | Toxiproxy/scripted | pre-event rehearsal |

## 2. Critical test cases (must exist before each related milestone ships)

### Unit / game-logic
- Phone normalization: Persian/Arabic/Latin digits; `09…`, `9…`, `+989…`, `00989…`; rejects landlines/non-Iranian.
- Name normalization: Arabic ی/ک, ZWNJ preserved, bidi controls stripped, length by graphemes.
- Each game: golden action logs → expected score (from SPEC examples, e.g. Perfect at combo 3 = 120; Fridge +100/+50 and multipliers; Vision speed bonus bands).
- Replay rejects: out-of-order timestamps, actions during pause/lock, taps on non-existent objects, placements of non-present items.
- `maxScore` ≥ any generated legal log's score (property-based test with random bots).
- PRNG: same seed → same sequence across engines.

### Integration
- Attempt limit under concurrency: 20 parallel `/start` calls with limit 1 → exactly 1 attempt.
- Duplicate result submission (parallel and sequential) → 1 attempt result, 1 base ticket, ≤ 1 grant per rule.
- Best score: sequence 6200 → 4100 → 7900 → best 7900; equal score keeps earlier `best_achieved_at`.
- Tie-break ordering with identical scores and timestamps.
- Reward inventory: 100 concurrent eligible results, `total_limit = 10` → exactly 10 grants; code pool of 5 → exactly 5 codes assigned, no duplicates.
- Outbox: result committed even when Snowa fake returns 500/timeouts; retries scheduled; lease recovery after process kill/restart.
- Draw: fixture population → expected eligible counts per filter; winners reproducible from stored seed; no duplicate winners; idempotent execute.
- Invalidation recomputes best/rank correctly.

### E2E (Playwright)
- AC-001 first-time journey; AC-002 returning; name screen cannot be skipped via URL.
- Disabled game visible and not startable (AC-005).
- Attempt exhaustion UI and server refusal (AC-006).
- Refresh during pre-game reuses session; refresh during play → interrupted state; refresh after end → pending result resubmitted.
- Offline during gameplay (network emulation) → result submitted after reconnect.
- Public display never contains phone digits pattern (AC-017).
- No Latin UI words outside allowlist (AC-003).

## 3. Test data and environments

- Seeded fixtures: participants with varied progress, rewards with limits, draw populations of known shape.
- Fake OTP sender in dev/staging: codes visible in a dev-only endpoint/log, **disabled in production builds by config guard and startup check**.
- Fake Snowa receiver with programmable behavior (success, 500, timeout, 429, 400, slow).
- Time control: server clock abstraction in tests (DB `now()` via injectable clock for unit tests; real DB time in integration tests with short configured windows).

## 4. Quality gates

| Gate | Criteria |
|---|---|
| PR | lint (incl. RTL logical-properties rule, no hard-coded strings), typecheck, unit, integration, contract tests green |
| Main | E2E green on Chromium + WebKit |
| Milestone | acceptance criteria for scope verified; device matrix smoke; load profile passed |
| Event readiness | full acceptance run, rehearsal, runbook drill, backups verified |
