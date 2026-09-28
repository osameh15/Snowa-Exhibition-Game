# Participant Management and Audit History

Source: SPEC §16.3, §21, OPS-001.

## 1. Participant search

| Query | Matching |
|---|---|
| Name | trigram similarity on normalized display name |
| Phone | exact (normalized E.164) or last-4 suffix; results show masked phone |
| Filters | status, has flagged attempts, completed all games, winner of draw X, has reward Y |

Results: keyset paginated; masked phones; no bulk phone display.

## 2. Participant detail

| Section | Content | Actions (permission) |
|---|---|---|
| Profile | name, masked phone, status, verified at, last seen, sessions count | reveal phone (`view_phone`, audited), hide/edit name, block/unblock |
| Per-game progress | attempts used/allowed (incl. bonus), best score, rank, first completion, base ticket | grant bonus attempt, exclude from ranking |
| Attempts | number, status, claimed vs validated score, flags, active/paused ms, device class, timestamps | open attempt detail, invalidate, clear flags, restore |
| Tickets | source, game/reward, status | void, reinstate, grant |
| Rewards | grants with status/code (masked for Operator) | fulfillment update, revoke |
| External deliveries | per attempt: status, attempts, last error | retry |
| Draws | eligibility/wins | — |
| Audit excerpt | actions targeting this participant | — |

## 3. Attempt detail (review tool)

Shows payload summary, recomputed stats, flags with explanations, timing chart (tap/reaction distribution), seed/config version, and a "replay visualization" link (Phase 7 nice-to-have: re-run the action log in a debug build of the game).

Invalidate dialog: reason (required, min 10 chars), impact preview (best score before/after, rank change, ticket handling per OQ-23, external correction if supported).

## 4. Manual changes policy

- No direct score editing (history must stay truthful). Allowed corrections: invalidate/restore attempts, bonus attempts, ticket void/grant, ranking exclusion — each with elevated permission, reason, immutable audit [SPEC §16.3].
- Whether any manual correction is permitted during the event is confirmed by OQ-31; technically they can be disabled per event policy.

## 5. Audit history screen

Filters: time range, actor, role, action, target type/id. Row: time (Jalali, Tehran), actor, action (Persian label), target link, reason, before/after diff (masked PII). Export (Super Admin). Hash-chain status indicator (last verification time, OK/broken).
