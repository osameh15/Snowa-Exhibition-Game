# Audit Model

Source: SPEC §15.2, §16 ("critical actions require confirmation and create audit records"), §16.3, §21, OPS-001.

## 1. What is audited

Audit log = **who did what to which object, when, from where, why, with before/after**. It is separate from analytics and from application logs.

| Category | Actions (examples, `action` codes) |
|---|---|
| Admin identity | `admin.login_succeeded`, `admin.login_failed`, `admin.totp_failed`, `admin.logout`, `admin.user_created`, `admin.role_changed`, `admin.password_reset`, `admin.disabled` |
| Event & games | `event.status_changed`, `event.policy_changed`, `game.state_changed`, `game.emergency_stop`, `game.attempt_limit_changed`, `game.config_version_published` |
| Rewards | `reward.definition_created/updated`, `reward.rule_created/updated/activated/paused`, `reward.codes_uploaded`, `reward.grant_revoked`, `reward.fulfillment_updated` |
| Participants | `participant.viewed_phone` (full phone reveal), `participant.blocked/unblocked`, `participant.name_hidden/edited`, `participant.bonus_attempt_granted` (records participant, game, actor, reason, timestamp, related `game_session_id`/`attempt_id` when applicable, count), `participant.erased` |
| Scores & tickets | `attempt.invalidated`, `attempt.restored`, `attempt.flags_cleared`, `ticket.voided/reinstated`, `ticket.admin_granted`, `progress.ranking_excluded` |
| Raffle | `draw.created`, `draw.previewed`, `draw.execute_requested`, `draw.executed`, `draw.presented`, `draw.revealed`, `draw.voided`, `draw.cancelled` |
| Integration | `delivery.retry_requested`, `delivery.bulk_retry`, `delivery.marked_resolved`, `integration.config_changed` |
| Data | `export.created` (what, filters, row count), `display.token_created/revoked` |
| System | `system.integrity_mismatch_detected`, `system.retention_purge_run` |

Participant self-actions (OTP, play) are recorded in their own transactional tables, not in `audit_log`, except security events (`participant.otp_locked`).

## 2. Record structure

| Field | Type | Notes |
|---|---|---|
| `id` | bigint identity | monotonic |
| `occurred_at` | timestamptz | DB time |
| `actor_type` | `ADMIN` · `SYSTEM` · `DISPLAY` · `PARTICIPANT` | |
| `actor_id` | UUID/null | |
| `actor_role` | text | role at time of action |
| `action` | text | code from §1 |
| `target_type` / `target_id` | text / text | e.g., `game_settings` / `spin-perfect` |
| `reason` | text/null | mandatory for high-risk actions |
| `before` / `after` | jsonb | changed fields only; PII minimized (masked phone) |
| `request_id` | text | correlates with logs |
| `ip_hash` / `user_agent` | text | IP stored as keyed hash (privacy) plus coarse network prefix |
| `prev_hash` / `row_hash` | bytea | optional hash chain (SHA-256 over previous hash + row) for tamper evidence |

## 3. Integrity rules

- Append-only: the application DB role has `INSERT, SELECT` only on `audit_log`; no `UPDATE/DELETE`. Migrations run under a separate role.
- Audit insert happens **in the same transaction** as the audited change (both commit or neither). Exception: `draw.execute_requested` and `admin.login_failed` are committed independently (they must survive a rolled-back action).
- Hash chain (recommended): computed by a `BEFORE INSERT` trigger using the latest row hash under an advisory lock; verification job checks continuity daily. Cost is negligible at admin-action volumes.
- Retention: at least as long as the longest retained related data, and not shorter than the prize-dispute window (OQ-13).

## 4. Access

| Role | Audit access |
|---|---|
| Operator | Own actions only |
| Administrator | All event actions; PII masked |
| Super Admin | All, export |
