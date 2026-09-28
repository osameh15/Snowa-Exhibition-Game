# Fridge Rush — Technical Gameplay Specification

| | |
|---|---|
| Internal id | `fridge-rush` (games.id = 2) |
| Working Persian title | چیدمان سریع (brand may rename) |
| Hero product | Snowa refrigerator (open, 2.5D interactive board) |
| Mechanic | Quick recognition and placement: drag items from a tray into the correct refrigerator zone |
| Duration | 50 000 ms [SPEC §10.1] |
| Phase | Phase 3 |

Product behavior preserved from SPEC §10. *Tunable* = config parameter.

## 1. Board model

| Element | Definition |
|---|---|
| Zones (4–5) | `freezer`, `main_shelf`, `door_rack`, `crisper_drawer`, optional `deli_drawer` [SPEC §10.2]. Each zone: id, polygon in stage coordinates (for client hit-testing), Persian label + icon (signposting) |
| Item types (8–10) | e.g. `milk`, `juice`, `water`, `apple`, `vegetables`, `cake`, `ice_cream`, `eggs`, `cheese`, `meat` [SPEC §10.3]. Each: id, art key, `correctZones[]` (v1: exactly one; config allows list) |
| Tray | `slots` visible item slots at the bottom (thumb zone); each slot holds one item instance |
| Item instance | `(instanceId = spawn counter, type, slot, presentedAt)` — deterministic from seed and history |

Zone-to-item mapping is config (not code) and must be "intuitive, arcade-like" — reviewed by product with the art.

## 2. Spawn rules (deterministic)

- At `t = 0`: fill `slots(phase)` slots with instances from the PRNG type sequence (no three identical types in a row; each type appears at least once per 10 spawns).
- When an instance is placed **correctly**, its slot refills after `spawnDelayMs = 250` (*tunable*) with the next PRNG type → `presentedAt = t_place + spawnDelayMs`.
- Wrong placement: item returns to its slot, remains actionable, `presentedAt` unchanged.
- Phase increases add slots (new slots fill at the phase boundary).
- Optional per-item timers (`itemTimeoutMs`, default *disabled*) — if enabled, expired items are replaced without penalty.

Because spawns depend only on seed and correct-placement times, the server reconstructs the exact tray at any `t` from the action log.

## 3. Difficulty schedule (*tunable*)

| Phase | Time | Slots | Notes |
|---|---|---|---|
| P1 | 0–15 s | 3 | distinct item categories |
| P2 | 15–35 s | 4 | visually closer items appear |
| P3 (final rush) | 35–50 s | 5 | faster spawn (`spawnDelayMs = 150`) |

## 4. Interaction

- Pointer down on a tray item → `grab` (item lifts, drop zones highlight with shape + label).
- Drag follows finger with offset so the item is visible above the finger.
- Release over a zone polygon (expanded hit area ≥ 12 dp beyond visual edge) → `place(item, zone)`. Snap animation.
- Release outside all zones → item returns, **no penalty**, not logged as place (logged as `drop_none` for analytics only).
- One active drag at a time (first pointer only).
- `touch-action: none` on stage; no page scroll during drag [SPEC §20.3].
- Minimum drop-zone size: ≥ 64 dp in the shorter dimension on 360-px-wide screens [SPEC §28].

## 5. Scoring, combo, rush

```text
on place(item, zone, t):
  if zone in item.correctZones:
      streak += 1; combo += 1
      speed = t - item.presentedAt
      bonus = 50 if speed < 1000 else 30 if speed < 2000 else 15 if speed < 3000 else 0
      comboMult = 1500 if combo >= 8 else 1250 if combo >= 5 else 1100 if combo >= 3 else 1000
      rushMult  = rushActive(t) ? params.rushMultiplier (default 1500) : 1000
      points = floor((100 + bonus) * comboMult / 1000 * rushMult / 1000)
      score += points
      if not rushActive(t) and streak >= params.rushTrigger (default 5):
          rushUntil = t + params.rushDurationMs (default 5000); streak = 0
  else:
      score = max(0, score - 20)          # floor at 0 — OQ-19
      combo = 0; streak = 0
```

Values from SPEC §10.4 (+100 base; <1 s +50, <2 s +30, <3 s +15; −20 wrong; combo 3 ×1.1, 5 ×1.25, 8+ ×1.5) and §10.5 (rush after ~5 correct, ~5 s, temporary multiplier — trigger/multiplier are tuning parameters, not business rules). Multiplier applies to base + speed bonus (*tunable* decision confirmed in prototype).

## 6. Mystery bonus object (SPEC §10.6)

v1: optional **cosmetic** branded object may appear (seeded time/slot). Tapping it plays an effect and logs `[t,"bonus",id]`. It has **no scoring or reward effect**. Rewards are decided only by server rules at result time; the result screen may reveal an extra reward "from the mystery item" only if the server granted one. (OQ-20 for any future semantics.)

## 7. Timers, pause, end

- 50 s timer; final 5 s emphasis; no early game over.
- Pause hides the tray and fridge contents; an active drag is cancelled (item returns, no penalty).
- At `t = durationMs`, an in-progress drag is cancelled.

## 8. Result payload

```json
{ "actions": [[640,"grab",0],[1210,"place",0,"door_rack"],[1302,"grab",1],[1890,"place",1,"freezer"]],
  "stats": { "correct": 34, "wrong": 3, "maxCombo": 12, "rushCount": 4 } }
```

## 9. Score validation

| Check | Type | Rule |
|---|---|---|
| Replay | hard | Rebuild tray timeline; every `grab`/`place` must reference an instance present at `t`; `place` must follow its `grab` with no other grab in between |
| Drag duration | hard | `place.t − grab.t ≥ 80 ms` (*calibrate*; physically impossible below) |
| Placement interval | hard | consecutive `place` ≥ 120 ms apart |
| Ceiling | hard | `score ≤ maxScore(params, seed)` — simulate instant correct placement at each earliest legal time |
| Placement rate | flag | sustained > 2.5 correct/s over any 10 s window → `INPUT_RATE_HIGH` (*calibrate*) |
| Speed-bonus rate | flag | > 80 % placements with `< 1 s` bonus over ≥ 30 placements → `REACTION_TOO_FAST` (*calibrate*) |
| Near ceiling | flag | `≥ 0.95 × maxScore` |

## 10. Assets, audio, animation

| Group | Items |
|---|---|
| critical | open fridge board (layered: body, shelves, doors), zone overlays (idle/valid/invalid), 8–10 item sprites (atlas, readable at 56 px), tray, drag shadow |
| deferred | rush-mode VFX (frame glow, speed lines), combo effects, mystery object, confetti for result |
| audio | grab, correct (pitch rises with combo), wrong, rush start/end, final seconds, time-up |
| animations | snap-in, return-to-tray, zone pulse, +points pop, rush overlay |

## 11. Performance & accessibility

- Drag uses transform updates only (no re-layout); item sprite pool.
- Zone highlight uses shape outline + label + color [SPEC §28].
- Items must remain recognizable at small sizes (art review at 360×640).
- Left-handed/right-handed: tray centered; no side-dependent affordances.
