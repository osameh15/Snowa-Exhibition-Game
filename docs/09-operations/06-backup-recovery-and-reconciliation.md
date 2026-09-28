# Backup, Recovery and Reconciliation

## 1. Backup

| Item | Method | Frequency | Retention |
|---|---|---|---|
| PostgreSQL | Continuous WAL archiving (PITR) + daily base backup (managed service equivalent) | continuous | 30 days rolling + event-final snapshot |
| Pre-draw snapshot | Logical backup of draws/tickets tables (optional extra) | before major draws | event + 1 year |
| Config & secrets | Secret store backups / encrypted file in secure vault | on change | per policy |
| Static builds & images | Registry + artifact storage | per release | ≥ 90 days |

Backups encrypted at rest, stored in a separate location/account from the primary, access limited to DevOps.

## 2. Recovery objectives (proposed, approval needed)

| Objective | Target |
|---|---|
| RPO (data loss) | ≤ 1 min (WAL shipping/streaming replica) |
| RTO (service restore) | ≤ 30 min (replica promotion / managed failover) |

Restore drill before the event: restore last backup into staging and run integrity reconciliation + draw verification on restored data.

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
