# Acceptance Strategy

## 1. Acceptance criteria → verification

| AC [SPEC §29] | Verification | Automated | Milestone |
|---|---|---|---|
| AC-001 First-time QR→OTP→name→lobby | E2E + device matrix | ✓ | M1 |
| AC-002 Returning skips name | E2E | ✓ | M1 |
| AC-003 Persian/RTL only | E2E DOM scan + copy sign-off | ✓ + manual | M1, M7 |
| AC-004 Lobby 3 stable cards + state | E2E + visual regression | ✓ | M1 (1 game), M4 |
| AC-005 Disabled game visible, not startable | E2E + API test | ✓ | M2 |
| AC-006 Server blocks after limit | Integration (concurrency) + E2E | ✓ | M1 |
| AC-007 Lower later score doesn't reduce best | Integration | ✓ | M1 |
| AC-008 Every accepted attempt auditable | Integration (history + payload) | ✓ | M1 |
| AC-009 One base ticket per game | Integration + E2E | ✓ | M1 |
| AC-010 Extra rewards in addition | Integration (RW tests) | ✓ | M2 |
| AC-011 Result shows authoritative values | E2E (compare to API) | ✓ | M1 |
| AC-012 Best-score ranking + tie-break | Integration | ✓ | M1 |
| AC-013 Admin toggles/limits without deploy | E2E admin | ✓ | M2 |
| AC-014 Extra rewards without removing core | E2E + API | ✓ | M2 |
| AC-015 Preview + execute raffle with filters | E2E + RF tests | ✓ | M5 |
| AC-016 Server-side, persisted, auditable draw | RF-10/13/14/17 | ✓ | M5 |
| AC-017 No full phones on public display | E2E regex scan of display/public payloads | ✓ | M2, M5 |
| AC-018 Persist when Snowa down; retry later | F2 injection | ✓ | M1 (fake), M6 (real) |
| AC-019 Duplicate submission safe | Integration concurrency | ✓ | M1 |
| AC-020 Works without install | E2E + device matrix | ✓ + manual | M1 |
| AC-021 Games usable portrait mobile/tablet | Device matrix | manual | M1/M3/M4, M7 |
| AC-022 Live views recover after disconnect | F3/F9/F13 | ✓ | M2, M5 |

## 2. Event rehearsal (Milestone 7 exit)

A full dress rehearsal in the staging environment configured like production:
1. ≥ 30 staff with their own phones play all three games; one simulated QR booth.
2. Load generator runs L2 in parallel.
3. Operators perform: toggle game, change attempts, create/activate reward, emergency stop/clear, participant lookup, bonus attempt, flagged review.
4. Run a live draw on the booth display with presentation/reveal.
5. Inject F1, F2, F3 during the rehearsal.
6. Verify reports and deliveries reconcile.
7. Sign-off by product, operations and engineering; open defects triaged.

## 3. Sign-off artifacts

Test reports, device matrix results, load test report, security checklist, rehearsal notes, runbook drill log, backup-restore proof.
