# Raffle and Reward Integrity

## 1. Raffle integrity controls

| Requirement [SPEC §15, §21] | Control |
|---|---|
| Server-side selection | Selection runs only in the API execution transaction; displays receive persisted results after reveal |
| Unbiased randomness | CSPRNG seed (32 bytes) + HMAC-SHA256 stream + rejection sampling; algorithm versioned |
| Reproducibility | Seed, snapshot entries, snapshot hash, algorithm version stored; `verify-draw` tool recomputes |
| Snapshot freeze | Eligibility materialized into `draw_entries` in the execution transaction |
| No silent re-roll | No re-run on a completed draw; void requires Super Admin + reason; new draw references the voided one |
| Attempted executions visible | `draw.execute_requested` audit committed before selection |
| Double click / concurrent admins | Idempotency-Key + row lock + state check |
| Early leak via display | Display API returns only revealed positions |
| Tamper evidence | Write-once trigger on completed draws; append-only hash-chained audit |
| Eligibility correctness | Criteria schema-validated; preview shows counts; integration tests on known fixtures |
| Legal rules (eligibility restrictions, staff exclusion, duplicate winners across draws) | Business/legal decision (OQ-10, OQ-29); technically supported by criteria + exclusion lists |

Optional (recommended for high-value prizes): display the `snapshotHash` on the public screen at presentation start, and publish the seed afterward, so any third party can verify with the exported pseudonymized entries.

## 2. Reward integrity controls

| Risk | Control |
|---|---|
| Client decides reward | Evaluation only server-side in result tx; games have no reward logic [SPEC §10.6] |
| Overspend of inventory | Conditional `granted_count < total_limit` update; code pool `SKIP LOCKED`; unique code→grant |
| Duplicate grants on retry | `(rule_id, source_attempt_id)` unique; session single result |
| Unlimited expensive reward by mistake | Server rule validation + Super Admin for unlimited + typed confirmation |
| Probability manipulation | CSPRNG rolls recorded in evaluation trace |
| Insider self-grant | Manual grants require Super Admin + reason + audit; manual-action report |
| Code leakage | Codes visible only to recipient and authorized admins; no bulk display to Operators; exports audited |
| Replayed old rule | Grants store `rule_snapshot`/version for audit |
