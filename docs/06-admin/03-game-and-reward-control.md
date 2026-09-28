# Game Control, Attempt Configuration and Reward Management

Source: SPEC §16.2, §16.4, §13, AC-013, AC-014.

## 1. Game control

| Control | Behavior | Live effect | Audit |
|---|---|---|---|
| Enable/disable toggle | Per game; disabled games stay visible to participants as unavailable | New sessions immediately (≤ 2 s cache, DB re-check on create/start) | `game.state_changed` |
| Emergency stop | Red control per game + event-wide "pause event"; reason required | Blocks new sessions and ISSUED starts; STARTED may submit (flagged) | `game.emergency_stop` |
| Attempt limit | Stepper 1–100; shows impact preview before save ("X participants will have no attempts left", "Y regain attempts") | New session creation/start | `game.attempt_limit_changed` |
| Core reward | Read-only card "بلیط قرعه‌کشی — 1 per first completion" (SPEC §13.2); editable only via Event settings by Super Admin | — | `event.policy_changed` |
| Extra rewards | Link to rules filtered by game; add rule | ≤ 2 s | reward audit |
| Metrics | players, accepted attempts, avg/best score, flagged count, recent activity | live | — |
| Config version | shows active version; publishing new versions is Super Admin only and blocked while LIVE (requires force + reason) | — | `game.config_version_published` |

Concurrency: every change sends `version`; conflicting concurrent edits get `409 VERSION_CONFLICT` and the UI reloads the latest state with a Persian notice.

## 2. Reward management

### Definition editor
Fields: type, fulfillment mode, Persian title/description/terms (RTL preview as participant would see it on result screen), shared code or code pool, ticket quantity (extra tickets), validity date.

### Rule editor
Fields: definition, trigger, games, score thresholds, probability (entered as %), active window (Jalali date-time picker, Tehran time), total limit, per-participant limit, exclusive group, priority.

Safety UX [SPEC §16.4]:
- Physical prize / benefit: total limit field mandatory.
- Unlimited: requires Super Admin + explicit checkbox "I understand this reward is unlimited" + typed confirmation.
- Pre-activation summary: "At current rates (~N valid results/hour), this rule would grant ~M rewards/hour; inventory lasts ~H hours" (computed from last hour of accepted attempts).
- Activation requires confirmation dialog listing: games, trigger, probability, limits, window.
- Low inventory alert at ≤ 10 % or ≤ 10 units (`reward_inventory_low`).

### Code pools
CSV upload (one code per line, ≤ 50k), trimmed, deduped (report duplicates/invalid), stock view (available/assigned/void). Codes are never displayed in bulk to Operators.

### Grants & fulfillment
List/filter by status, definition, participant. Physical prize workflow: `CLAIM_PENDING → CLAIM_CONTACTED → FULFILLED` with note, handler and time; revoke with reason (optionally return counted inventory).
