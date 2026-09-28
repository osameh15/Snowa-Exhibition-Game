# Admin Roles and Permissions

Source: SPEC §3. Roles are fixed in code (no role editor in v1); assignment by Super Admin. Authentication method is **assumed** local accounts + TOTP (OQ-21).

## 1. Roles

| Role | Purpose [SPEC §3] |
|---|---|
| `OPERATOR` | Booth staff: monitor, look up participants, run allowed operational actions |
| `ADMIN` (Event Administrator) | Event configuration, rewards, draws, exports, audit |
| `SUPER_ADMIN` | Everything + integrations, admin management, emergency overrides, event-level settings |
| Display (device) | Read-only public projection; not a human role |

## 2. Permission matrix

| Permission | OPERATOR | ADMIN | SUPER_ADMIN |
|---|---|---|---|
| `dashboard.view`, `leaderboard.view`, `game.view`, `draw.view`, `reward.view`, `integration.view`, `report.view` | ✓ | ✓ | ✓ |
| `participant.view` (masked phone) | ✓ | ✓ | ✓ |
| `participant.view_phone` (reveal full, audited) | — | ✓ | ✓ |
| `attempt.view` | ✓ | ✓ | ✓ |
| `participant.grant_bonus_attempt` | ✓ (max 1 per participant+game, reason) | ✓ | ✓ |
| `participant.moderate_name` | ✓ | ✓ | ✓ |
| `participant.block` | — | ✓ | ✓ |
| `participant.erase` | — | — | ✓ |
| `game.toggle` | — | ✓ | ✓ |
| `game.emergency_stop` | ✓ (stop only) | ✓ (stop + clear) | ✓ |
| `game.set_attempt_limit` | — | ✓ | ✓ |
| `game.publish_config` | — | — | ✓ |
| `event.manage` | — | — | ✓ |
| `reward.manage`, `reward.upload_codes` | — | ✓ | ✓ |
| `reward.create_unlimited` | — | — | ✓ |
| `reward.fulfill` | ✓ | ✓ | ✓ |
| `attempt.invalidate` (incl. clear flags) | — | ✓ | ✓ |
| `attempt.restore` | — | — | ✓ |
| `ticket.void` | — | ✓ | ✓ |
| `ticket.grant` / reinstate | — | — | ✓ |
| `leaderboard.exclude` | — | ✓ | ✓ |
| `draw.manage` (create/preview/cancel) | — | ✓ | ✓ |
| `draw.execute` | — | ✓ | ✓ |
| `draw.present` (present/reveal) | ✓ | ✓ | ✓ |
| `draw.void` | — | — | ✓ |
| `display.manage` | ✓ (mode only) | ✓ | ✓ |
| `data.export` | — | ✓ (phones only with `view_phone`) | ✓ |
| `audit.view_own` / `audit.view_all` | own | all (masked) | all |
| `integration.retry` | — | ✓ | ✓ |
| `integration.bulk_retry`, `integration.configure` | — | — | ✓ |
| `admin.manage` | — | — | ✓ |

The matrix is an assumption for approval (the SPEC gives role purposes, not exact permissions). Enforcement: server-side per endpoint; UI hides unavailable actions but never relies on hiding.

## 3. Admin account rules

- Unique named accounts (no shared logins), including booth staff.
- Password ≥ 12 chars, argon2id; TOTP mandatory for all roles (assumption A-12, OQ-21).
- Idle timeout: 30 min (Operator 8 h on dedicated booth devices if approved); absolute session 12 h.
- Lockout: 5 failed logins → 15 min lock + audit + alert.
- Optional IP allowlist for `admin.<domain>` (venue + office ranges).
- Super Admin accounts: ≤ 3, used only for sensitive operations.
