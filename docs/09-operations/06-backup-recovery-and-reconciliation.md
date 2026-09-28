# Backup, Recovery and Reconciliation

## 1. Backup

| Item | Method | Frequency | Retention |
|---|---|---|---|
| PostgreSQL — baseline (staging + production) | `pg_dump -Fc` via systemd timer, encrypted (e.g., age/gpg), uploaded **off-server** (object storage or another host; vendor TBD) | daily | 30 days rolling + event-final snapshot |
| PostgreSQL — event production (recommended) | Additionally hourly dumps during event days, **or** continuous WAL archiving (e.g., pgBackRest/WAL-G to off-server storage) for point-in-time recovery | hourly / continuous | event + 30 days |
| Pre-deploy / pre-draw dump | On-demand `pg_dump` before deploys and major draws | on demand | event + 1 year (draw dumps) |
| VPS snapshot (if provider offers) | Provider snapshot after setup and before event | on demand | until after event |
| Config & secrets | Encrypted copies in the team vault | on change | per policy |
| Release artifacts | CI artifact storage (web + server archives) | per release | ≥ 90 days |

Backups are encrypted before leaving the VPS, stored off-server under a separate account/credential, and access-limited to the operators. Backup success is monitored (alert if the latest successful off-server backup is older than its schedule).

Result durability on the live system does not depend on backups: every accepted result is committed with PostgreSQL defaults (`fsync = on`, `synchronous_commit = on`) before the participant sees it.

## 2. Recovery objectives

Proposed targets (operations approval needed):

| Objective | Staging | Event production |
|---|---|---|
| RPO (data loss if the VPS disk is lost) | ≤ 24 h | ≤ 1 h with hourly dumps; ≤ 5 min with WAL archiving (recommended) |
| RTO (service restore) | best effort | ≤ 60 min to rebuild a VPS from the setup runbook and restore; minutes for process/DB restarts |

Restore drill before the event: provision a fresh VPS from the setup runbook, restore the latest off-server backup, run integrity reconciliation and draw verification on the restored data, and record the measured RTO.

## 3. Integrity reconciliation (internal)

`integrity.reconcile` job (10 min + on demand), results in Admin → Integrations/System:

| Check | Query idea | On mismatch |
|---|---|---|
| Best score materialization | progress.best_score vs MAX(valid attempts) | alert; admin "recompute" action (audited) |
| Base tickets | first valid completion exists ⇔ BASE ticket exists | alert; admin fix via grant/void |
| Attempts used | progress.attempts_used vs COUNT(attempts) | alert |
| Reward counts | rule.granted_count vs COUNT(grants) (non-revoked, per return policy) | alert |
| Outbox coverage | valid attempts without delivery | enqueue missing (automatic, logged) |
| Draw verification | recompute winners for completed draws | alert Sev-1 on mismatch |
| Audit hash chain | continuity | alert Sev-1 |

## 4. External API reconciliation

1. Daily (and at event close): Admin → Integrations report of delivered/failed/pending by day and game.
2. If Snowa provides a received-records export or query API (TBD, OQ-04), compare by delivery id / phone+game+score.
3. Discrepancies: re-enqueue via replay tool (Super Admin, audited) — safe only if Snowa is idempotent; otherwise coordinate manually with Snowa.
4. Final sign-off: zero PENDING/RETRY_SCHEDULED; FAILED resolved or documented.
