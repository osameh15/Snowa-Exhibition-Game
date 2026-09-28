# Degraded Behavior and Failure Handling

Principle [SPEC §22.1]: fail safely, never lose accepted gameplay data, never duplicate attempts/tickets/rewards, never show a winner before persistence, show stale/reconnecting states honestly.

| Failure | Detection | Participant experience | Admin/display | Data safety | Recovery |
|---|---|---|---|---|---|
| OTP provider slow | send p95 > 5 s | Spinner then "SMS may be delayed" + resend after cooldown | OTP health amber | Challenges valid for TTL | Automatic |
| OTP provider down | send failures | "Code could not be sent, retry" (Persian); existing sessions play normally | Alert Sev-1; health red | No participant rows created without verify | Retry; secondary provider if configured |
| Snowa API down | delivery failures | None | Outbox backlog, degraded badge | Outbox durable | Auto retry; manual retry for FAILED |
| SSE disconnect | client heartbeat timeout | Leaderboard shows "updating…"; data refetched | Reconnecting badge; stale banner after 10 s; polling fallback | None (hints only) | Auto reconnect + snapshot |
| Refresh during gameplay | session STARTED without local payload | "This attempt was interrupted" + lobby | Attempt shows ABANDONED later | Attempt consumed per policy | Operator bonus attempt if justified |
| Refresh after game end | pending payload in storage | Result appears after resubmission | — | Idempotent | Automatic |
| Browser crash | same as refresh | same | — | same | same |
| PWA reopened later | session cookie valid | Lobby with current state | — | — | — |
| Backend restart (deploy or crash; single process) | health checks, systemd | Gap of a few seconds; idempotent calls retried; gameplay continues locally | SSE reconnect + `resync` | Committed tx and outbox rows safe; `IN_FLIGHT` deliveries recovered by lease expiry | systemd `Restart=always` |
| Full backend outage | health fails | Persian "connection problem" + retry; gameplay in progress continues locally; results held locally | Displays stale banner | Pending results in localStorage; late window 10 min | Resubmit on recovery |
| Duplicate result submission | session SUBMITTED | Same result shown | — | No duplicates | — |
| DB transient error | retryable error | Transparent retry (server) then client retry | — | Tx atomic | Automatic |
| Reward inventory exhausted | conditional update 0 rows | No extra reward shown (base ticket unaffected) | Low/zero inventory alert | No overspend | Admin adds inventory |
| Admin clicks raffle twice | idempotency/row lock | — | Same draw shown | Single draw | — |
| Live display disconnect | heartbeat | — | Display "reconnecting"; admin sees display offline | Reveal state persisted | Auto reconnect |
| High network latency | client timing | Loading states; gameplay unaffected; submission retries | — | Idempotent | — |
| Proxy/SW cache serving stale shell | build hash mismatch | Service worker update prompt on next navigation; API compatibility via `426` | — | Server validates versions | Purge cache; hashed assets |
| Stale game config in client | `configChecksum` mismatch | Session creation returns current config each time → never stale | — | Validation pinned to session version | — |
| Background jobs stalled (in-process `jobs` module) | outbox age, sweeper lag | None | Backlog grows | Outbox durable; sessions lazily expired on access | Restart Fastify; investigate job logs |
| Disk full / DB read-only | alerts | 503 on writes; lobby may still read | Alerts | No partial writes | Ops intervention |
| VPS failure (single point of failure) | uptime monitor | App unreachable; in-progress results kept in `localStorage` (late window) | Displays stale | Committed data on disk; off-server backups | Reboot; or rebuild VPS + restore backup (RPO/RTO in [Backup](06-backup-recovery-and-reconciliation.md#2-recovery-objectives)) |
