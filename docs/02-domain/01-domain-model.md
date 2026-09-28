# Domain Model

## 1. Aggregates and relationships

```mermaid
classDiagram
  direction LR
  class Event {
    id
    slug
    status
    timezone
    starts_at
    ends_at
    policies
  }
  class Participant {
    id
    phone_e164
    display_name
    status
    verified_at
  }
  class Game {
    id
    slug
    external_ref
  }
  class GameSettings {
    event_id
    game_id
    state
    attempt_limit
    active_config_version_id
    version
  }
  class GameConfigVersion {
    id
    game_id
    version
    params
    bounds
    checksum
  }
  class GameSession {
    id
    state
    seed
    issued_at
    start_by
    started_at
    deadline_at
  }
  class Attempt {
    id
    seq
    attempt_number
    status
    claimed_score
    attempt_score
  }
  class ParticipantGameProgress {
    attempts_used
    bonus_attempts
    best_score
    best_achieved_at
    first_valid_completion_at
  }
  class RaffleTicket {
    id
    source_type
    status
  }
  class RewardDefinition {
    id
    type
    title_fa
    fulfillment_mode
  }
  class RewardRule {
    id
    trigger
    conditions
    probability
    limits
    window
  }
  class RewardCode {
    id
    code
    status
  }
  class RewardGrant {
    id
    status
    fulfillment_state
  }
  class Draw {
    id
    state
    criteria
    seed
    eligible_count
  }
  class DrawEntry {
    draw_id
    participant_id
    weight
  }
  class DrawWinner {
    draw_id
    position
    participant_id
  }
  class ExternalDelivery {
    id
    status
    attempt_count
  }
  class AuditLog {
    id
    actor
    action
    target
    before
    after
  }

  Event "1" --> "*" GameSettings
  Game "1" --> "*" GameSettings
  Game "1" --> "*" GameConfigVersion
  Participant "1" --> "*" GameSession
  GameSession "1" --> "0..1" Attempt : started
  Participant "1" --> "*" ParticipantGameProgress : per game
  ParticipantGameProgress --> Attempt : best_attempt
  Attempt "1" --> "0..1" RaffleTicket : base ticket source
  RewardDefinition "1" --> "*" RewardRule
  RewardRule "1" --> "*" RewardGrant
  RewardDefinition "1" --> "*" RewardCode
  RewardGrant "0..1" --> "0..1" RewardCode
  RewardGrant "1" --> "*" RaffleTicket : extra tickets
  Draw "1" --> "*" DrawEntry
  Draw "1" --> "*" DrawWinner
  Attempt "1" --> "0..1" ExternalDelivery
```

## 2. Entity responsibilities

| Entity | Responsibility | Authority | Lifecycle doc |
|---|---|---|---|
| Event | Single exhibition config, status (`DRAFT/LIVE/PAUSED/CLOSED`), timezone, policies (core ticket policy, late-submission window, pause budget defaults) | Super Admin | this doc §4 |
| Participant | Verified phone identity + display name | identity module | [Participant lifecycle](02-participant-lifecycle.md) |
| Game / GameSettings | Catalog + live controls | catalog module | [Game/session/attempt](03-game-session-attempt-lifecycle.md) |
| GameConfigVersion | Immutable tuning & score bounds | catalog (published by Super Admin) | same |
| GameSession | Authorization envelope for one attempt | play module | same |
| Attempt | Gameplay record, validation outcome | play module | same |
| ParticipantGameProgress | Attempt counter, Best Score materialization, first completion marker | scoring module | [Best score & leaderboard](04-best-score-and-leaderboard.md) |
| RaffleTicket | Chance unit | tickets module | [Raffle tickets](05-raffle-tickets.md) |
| Reward* | Extra rewards | rewards module | [Rewards](06-rewards.md) |
| Draw* | Live raffle execution | raffle module | [Live raffle](07-live-raffle.md) |
| ExternalDelivery | Outbox | integration module | [Snowa adapter](../04-api/08-external-snowa-adapter.md) |
| AuditLog | Sensitive-action history | audit module | [Audit model](08-audit-model.md) |

## 3. Domain invariants (summary)

Full list with enforcement mechanism: [Invariants & transactions](../03-data/03-invariants-and-transactions.md).

| ID | Invariant |
|---|---|
| INV-01 | One participant per normalized phone per platform. |
| INV-02 | At most one open (`ISSUED` or `STARTED`) session per participant + game. |
| INV-03 | `attempts_used ≤ attempt_limit + bonus_attempts` at the moment each attempt started. |
| INV-04 | Attempt numbers per participant + game are unique and contiguous from 1. |
| INV-05 | A session produces at most one attempt; an attempt has at most one accepted result. |
| INV-06 | `best_score = max(attempt_score)` over valid attempts; null if none. |
| INV-07 | At most one `BASE` ticket per participant + event + game. |
| INV-08 | A single-use reward code is assigned to at most one grant. |
| INV-09 | Reward grants never exceed a rule's `total_limit` / `per_participant_limit`. |
| INV-10 | A completed draw's snapshot, seed and winners are immutable. |
| INV-11 | A participant appears at most once among a draw's winners. |
| INV-12 | Every valid attempt has exactly one external delivery record (unless integration disabled for the event). |
| INV-13 | Scores are integers ≥ 0. |

## 4. Event (exhibition) lifecycle

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> LIVE: Super Admin opens event
  LIVE --> PAUSED: pause (all new sessions blocked)
  PAUSED --> LIVE: resume
  LIVE --> CLOSED: close event
  PAUSED --> CLOSED
  CLOSED --> [*]
  note right of LIVE
    Participants can log in in any state except DRAFT-without-preview.
    New game sessions only when LIVE and game state ENABLED.
    Leaderboards/draws remain readable when CLOSED.
  end note
```

## 5. State machine index

| State machine | Location |
|---|---|
| Participant onboarding | [02-participant-lifecycle.md §2](02-participant-lifecycle.md#2-onboarding-state-machine) |
| OTP challenge | [02-participant-lifecycle.md §3](02-participant-lifecycle.md#3-otp-challenge-state-machine) |
| Game availability | [03-game-session-attempt-lifecycle.md §1](03-game-session-attempt-lifecycle.md#1-game-availability) |
| Game session | [03-game-session-attempt-lifecycle.md §2](03-game-session-attempt-lifecycle.md#2-game-session-state-machine) |
| Attempt | [03-game-session-attempt-lifecycle.md §3](03-game-session-attempt-lifecycle.md#3-attempt-state-machine) |
| Raffle ticket | [05-raffle-tickets.md §3](05-raffle-tickets.md#3-ticket-state-machine) |
| Reward grant | [06-rewards.md §5](06-rewards.md#5-reward-grant-state-machine) |
| Live raffle (draw) | [07-live-raffle.md §4](07-live-raffle.md#4-draw-state-machine) |
| External delivery | [../04-api/08-external-snowa-adapter.md §5](../04-api/08-external-snowa-adapter.md#5-delivery-state-machine) |
| Event | this document §4 |
