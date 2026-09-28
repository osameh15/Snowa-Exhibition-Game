# Event-Day Operational Runbook (Architecture)

This is the runbook structure and its technical procedures; names, phone numbers and channels are filled in during Milestone 7.

## 1. Roles on event day

| Role | Responsibility |
|---|---|
| Incident lead (engineering) | Decisions on Sev-1/2, deploy approvals |
| Backend on-call | API/DB/worker/outbox |
| Frontend/game on-call | Client issues, device problems at booth |
| Event operator(s) | Admin panel, displays, participant support |
| Event administrator | Rewards, draws, attempt changes |
| Snowa contact | External API issues, prize/legal decisions |

## 2. Timeline

| When | Checklist |
|---|---|
| T-7 days | Load test passed; rehearsal done; backups + restore test; secrets rotated; admin accounts created with TOTP; displays provisioned; SMS sender template approved; Snowa production credentials verified with a test record (agreed with Snowa) |
| T-1 day | Change freeze; event record configured (dates, policies); game settings (ENABLED? attempts); reward rules reviewed (limits!); code pools uploaded; displays tokens set; QR codes tested on site network; SMS balance/quotas confirmed |
| T-2 h | Health dashboard green; test participant journey on 3 devices (Android/iOS/tablet) against production; display modes set; event status → LIVE |
| During | Watch dashboard; respond to alerts; operators review flagged top ranks hourly |
| Before each draw | Review flagged attempts in relevant ranks; confirm criteria with event admin; preview; execute; present |
| Close | Event status → CLOSED (or PAUSED overnight); verify outbox drained; export final reports; snapshot backup |
| T+1 day | Reconciliation with Snowa; incident review; retention actions scheduled |

## 3. Standard procedures

| Procedure | Steps |
|---|---|
| Pause everything (critical bug) | Admin → Event → PAUSED (reason). New sessions stop; in-flight results accepted. Communicate at booth. |
| Stop one game | Admin → Games → Emergency stop (reason). |
| Participant says "my game broke" | Look up by phone → check attempt status (ABANDONED/REJECTED?) → if genuine technical interruption, grant bonus attempt (reason). |
| OTP not arriving | Check OTP health panel; if provider degraded → announce delay; if down > 10 min → switch to secondary provider if configured (ADR-005) or pause onboarding messaging. |
| Snowa API down | Nothing blocks; monitor backlog; notify Snowa contact; after recovery confirm drain. |
| Reward out of stock | Alert received → decide: add codes / raise limit / pause rule. |
| Offensive display name | Participant detail → hide name (reason). |
| Display frozen/stale | Check display status in admin; reload display browser; re-auth with token if needed. |
| Draw executed by mistake | Do not re-run silently. Super Admin voids with reason; create new draw; communicate. |
| Suspected cheater at top | Review attempt detail; invalidate (reason) and/or exclude from ranking; keep evidence. |
| API replica failure | Proxy routes around; restart container; check error rate. |
| DB failover | Follow provider procedure; verify writes; watch late submissions. |

## 4. Communication templates

Persian booth messages (prepared by content owner) for: SMS delay, temporary pause, game unavailable, draw starting, winners announced.
