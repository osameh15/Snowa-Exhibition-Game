# Glossary

Terms are listed with their technical identifier (used in code, API and database) and meaning. Persian UI labels are defined in locale files, not here (starter set: SPEC Appendix A).

| Term | Identifier | Definition |
|---|---|---|
| Participant | `participant` | A person verified by phone + OTP. Identified internally by `participant_id` (UUID); phone is a unique attribute, not the primary key. |
| Display name | `display_name` | Free-text name entered after first OTP. Display data only; not identity verification [SPEC §6.3]. |
| Event | `event` | One exhibition configuration (dates, status, policies). v1 runs a single active event; all scoped data carries `event_id`. |
| Game | `game` | One of `spin-perfect`, `fridge-rush`, `vision-hunt`. Has a stable `slug`, numeric `id`, and per-event settings. |
| Game settings | `game_settings` | Live-editable per-event game controls: availability state, attempt limit, active config version. |
| Game config version | `game_config_version` | Immutable, versioned gameplay/scoring parameters (durations, windows, point tables, score bounds). Changing it requires deployment/approval, not a live toggle. |
| Game session | `game_session` | Server-issued authorization to play one attempt. States: `ISSUED → STARTED → SUBMITTED` or `EXPIRED/CANCELLED`. |
| Attempt | `attempt` | The gameplay record created when a session starts. Consumes one allowed attempt. Carries score, validation outcome and audit metadata. |
| Attempt number | `attempt_number` | 1-based sequence per participant + game, assigned at session start. |
| Attempt limit | `attempt_limit` | Admin-configured allowed attempts per participant per game (default 1) [SPEC PD-05]. |
| Valid attempt / valid completion | status `ACCEPTED` or `ACCEPTED_FLAGGED` | An attempt whose result passed server validation. Only these count for Best Score and base tickets. |
| Attempt score | `attempt_score` | Server-validated score of one attempt. |
| Claimed score | `claimed_score` | Score reported by the client; stored for audit, never authoritative. |
| Best Score | `best_score` | Max `attempt_score` over valid attempts for a participant + game. The official score [SPEC §12.1]. |
| Best achieved at | `best_achieved_at` | Server timestamp when the current Best Score was first accepted; tie-break key. |
| Rank | `rank` | 1-based position on a game leaderboard: best score desc, `best_achieved_at` asc, `best_attempt_seq` asc. |
| Leaderboard | `leaderboard` | Derived ranking view per game. Not a stored source of truth. |
| Raffle ticket | `raffle_ticket` | A persisted entitlement unit representing one chance/entry in raffles. Sources: `BASE`, `REWARD`, `ADMIN`. |
| Base ticket | source `BASE` | The core ticket granted once per participant per game on first valid completion [SPEC §13.1]. |
| Extra reward | `reward_grant` from a `reward_rule` | Admin-configured additional reward (discount code, extra ticket, physical prize, benefit, special). |
| Reward definition | `reward_definition` | What a reward is (type, Persian presentation, fulfillment mode). |
| Reward rule | `reward_rule` | When a reward is granted (trigger, conditions, probability, limits, window). |
| Reward inventory | `reward_rule.total_limit` / `reward_code` | Finite budget: counted cap and/or pool of single-use codes. |
| Reward grant | `reward_grant` | An issued reward to a participant, with fulfillment/redemption state. |
| Draw / Live raffle draw | `draw` | One raffle execution: frozen eligibility snapshot, seed, winners, audit data. Distinct from tickets. |
| Draw entry | `draw_entry` | A row of the frozen eligibility snapshot (participant, weight). |
| Draw winner | `draw_winner` | Selected participant for a draw, in selection order. |
| Reveal | `draw.revealed_count` | Controlled presentation of already-persisted winners on displays. |
| Public display | `display_device` | Read-only booth screen authenticated with a revocable display token. |
| Outbox / External delivery | `external_delivery` | Durable record of a payload to deliver to the Snowa API, with status and retry history. |
| Adapter | `SnowaResultAdapter` | Code that maps the canonical internal result to the external contract and performs transport. |
| Action log | `action_log` | Compact list of timestamped gameplay inputs submitted with a result; replayed server-side to recompute the score. |
| Seed | `seed` | Server-generated random value per session used by the deterministic PRNG for layouts/sequences. |
| Pause budget | `pause_budget_ms` | Total pause/background time allowed per attempt before the attempt auto-finishes. |
| Hard deadline | `deadline_at` | Server time after which a started session can no longer be submitted normally. |
| Flag | `attempt_flag` | A soft anomaly code on an accepted attempt (e.g., `LATE_SUBMISSION`, `REACTION_TOO_FAST`). |
| Audit log | `audit_log` | Append-only record of security- and operation-sensitive actions. |
| SSE | — | Server-Sent Events: one-way HTTP streaming from server to browser. |
| CSPRNG | — | Cryptographically secure pseudo-random number generator (`crypto.randomBytes`/`randomInt`). |
| E.164 | `phone_e164` | Canonical phone format, e.g. `+989123456789`. |
| Operator / Administrator / Super Admin | `OPERATOR`, `ADMIN`, `SUPER_ADMIN` | Admin roles [SPEC §3]; see [Roles](../06-admin/02-roles-and-permissions.md). |
