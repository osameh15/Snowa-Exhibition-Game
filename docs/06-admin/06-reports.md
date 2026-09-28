# Reports and Analytics (Admin)

Source: SPEC §23.2. Reports are computed from transactional tables (authoritative) and `analytics_events` (funnel only).

| Report | Content | Source | Export |
|---|---|---|---|
| Event overview | verified participants, attempts (valid/rejected/abandoned), completion rates, completed-all-three, tickets, rewards | transactional | CSV/XLSX |
| Funnel | QR entry → OTP requested → OTP verified → name → lobby → first game started → first completion → all three | analytics + transactional | CSV |
| Per-game performance | starts, completions, completion rate, score avg/median/p90/p99, histogram, replay rate, avg active time | attempts | CSV |
| Hourly activity | participants, attempts per hour per game (Tehran time) | attempts | CSV |
| Rewards | grants by definition/rule, inventory remaining, fulfillment status | rewards tables | CSV |
| Tickets | issued by source/game; distribution per participant | tickets | CSV |
| Draws | history with criteria, counts, winners (masked unless permitted) | draws | CSV |
| External delivery | daily delivered/failed/pending, error categories | outbox | CSV |
| Anti-cheat | flagged/rejected by code and game; top-rank review list | attempts/flags | CSV |
| OTP | requests, success rate, lockouts, provider failures | otp_challenges | CSV |

Rules:
- Exports are async jobs, audited (`export.created` with filters, row count), downloadable once within 15 min.
- Phone numbers included only if requested and the actor has `participant.view_phone`.
- Time columns: ISO UTC + Jalali Tehran.
- Report queries run against the primary with statement timeout 10 s; if load becomes an issue, move to a read replica (upgrade path).
