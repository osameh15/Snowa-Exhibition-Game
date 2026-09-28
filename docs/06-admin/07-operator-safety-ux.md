# Operator Safety and Confirmation UX

Source: SPEC §16 ("critical actions require confirmation"), §16.4, §16.5. Principle: the admin panel is used under pressure in a noisy booth; safety must not depend on operators reading carefully.

## 1. Risk tiers

| Tier | Examples | UX |
|---|---|---|
| T0 — read | viewing, filtering | none |
| T1 — reversible, low impact | display mode switch, reveal next, fulfillment note | single click; undo where possible; toast |
| T2 — reversible, participant-visible | enable/disable game, attempt limit change, pause reward rule, bonus attempt | confirmation dialog summarizing effect + impact preview |
| T3 — high impact | emergency stop, event pause/close, activate reward rule, invalidate attempt, void ticket, block participant, revoke grant | dialog + mandatory reason + explicit impact numbers |
| T4 — irreversible / fairness critical | execute draw, void draw, publish config version, unlimited reward, erase participant, bulk retry | dialog + reason (where applicable) + **typed confirmation** (e.g., type the winner count and event slug) + Super Admin where specified |

## 2. Rules

1. **No double execution**: buttons disable on submit; Idempotency-Key per dialog; server enforces anyway.
2. **Show state, not intent**: after any command the UI re-fetches and displays the server state (never assumes success).
3. **Stale-data guard**: every editable form carries `version`; conflicts show "another operator changed this" with the new values.
4. **Destructive controls separated**: emergency stop and draw execute are visually distinct (color + icon + position) and never adjacent to routine toggles.
5. **Clear environment marking**: staging/admin test environments show a persistent colored banner "محیط آزمایشی" so rehearsals are never confused with live.
6. **Live indicator**: header shows event status (LIVE/PAUSED/CLOSED) and connection state at all times.
7. **Persian, precise copy**: dialogs state the consequence in numbers ("۳ بازی... ۱٬۲۴۰ شرکت‌کننده…").
8. **Undo windows**: none for T3/T4 — corrections are new audited actions (restore, reinstate), never silent reverts.
9. **Keyboard/touch**: dialogs' default focus is "cancel" for T3/T4; confirm requires deliberate action.
10. **Audit visibility**: after T3/T4 actions, a toast links to the created audit entry.
