# Localization, RTL and Persian Typography

End-user UI: Persian only, RTL [SPEC §5]. Technical identifiers and documentation: English. Admin UI: Persian/RTL (bilingual is OQ-11).

## 1. Global rules

| Rule | Requirement |
|---|---|
| Document direction | `<html lang="fa" dir="rtl">` on participant and admin apps |
| CSS | Logical properties only (`margin-inline-start`, `padding-inline-end`, `inset-inline-start`, `text-align: start`). Physical `left/right` forbidden in shared components except for canvas/game geometry. Enforced via stylelint rule |
| Icons | Directional icons (back, next, chevrons, progress arrows) mirror in RTL; non-directional (play ▶ is conventionally not mirrored — follow design) specified per icon in the design system |
| Strings | All participant-facing text in `packages/i18n-fa` (keys from SPEC Appendix A). Build fails if a key is missing. No hard-coded strings in components (lint rule) |
| Production guard | E2E test scans rendered DOM for Latin-letter words outside an allowlist (brand marks, codes) [SPEC AC-003] |
| Server messages | API returns `error.code`; client maps to Persian. Server never returns user-facing prose |
| Admin-authored content | Reward titles/descriptions are entered in Persian by admins; stored as UTF-8 NFC |

## 2. Typography

- Persian web font: TBD with licensing (OQ-25). Candidates must support Persian glyphs, ZWNJ (U+200C), and both Persian and Latin digits. Self-hosted WOFF2, subset, `font-display: swap`.
- Line height ≥ 1.6 for body Persian text; avoid letter-spacing on Persian (breaks joining).
- Minimum body size 16 px on mobile (also prevents iOS input zoom).
- Truncation: `text-overflow: ellipsis` with `unicode-bidi: plaintext` on names; component widths designed for Persian first, not ported from English layouts [SPEC §24.1].
- Canvas text: in-game feedback words are pre-rendered bitmaps (see [Game runtime](04-game-runtime-architecture.md#6-hud-and-text-rendering)).

## 3. Mixed RTL/LTR content

Every LTR token inside Persian text MUST be bidi-isolated (`<bdi>` or `unicode-bidi: isolate` with `dir="ltr"`), otherwise punctuation and digits reorder visually.

| Content | Direction | Rendering rule |
|---|---|---|
| Phone number | LTR | `<bdi dir="ltr">0912 345 6789</bdi>`; grouped `4-3-4` for display; masked form `0912•••6789` |
| OTP input | LTR | Single `<input inputmode="numeric" autocomplete="one-time-code" dir="ltr">`; visual boxes optional; accepts Persian/Arabic/Latin digits |
| Scores | LTR number | Thousands separator per numeral policy; isolate |
| Timer | LTR `mm:ss` or `ss` | Isolate; tabular numerals to avoid jitter |
| Rank | number | `#42` rendered as Persian label + isolated number |
| Discount codes | LTR | Monospace/tabular, `dir="ltr"`, copy button; never transformed by digit localization |
| Identifiers (draw id, support ref) | LTR | Isolate; never localized |
| Participant names | auto | `dir="auto"` + `unicode-bidi: plaintext` (names may be Latin) |

## 4. Numerals (OQ-07 — product decision required)

SPEC §5.1 allows Latin digits for scores/timers if approved by UI design; concept art uses Latin digits for scores, ranks, phones and timers.

**Recommended default (pending approval):** single helper `formatNumber(value, context)` driven by a config map:

| Context | Recommended glyphs | Rationale |
|---|---|---|
| score, timer, rank, combo | Latin | Matches concept art; legibility at large sizes |
| phone, OTP, discount code, ids | Latin | Copy/paste and SMS consistency |
| counts inside Persian prose (e.g., "۲ بلیط") | Persian | Natural reading |
| admin tables | Latin | Sorting/scanning; configurable |

Changing the policy MUST require only config changes in `i18n-fa`, not component edits.

## 5. Input normalization (client and server)

| Input | Normalization (server is authoritative) |
|---|---|
| Digits | Map `۰-۹` (U+06F0–06F9) and `٠-٩` (U+0660–0669) → `0-9` |
| Phone | Strip spaces, dashes, parentheses; accept `09xxxxxxxxx`, `9xxxxxxxxx`, `+989xxxxxxxxx`, `00989xxxxxxxxx`, `989xxxxxxxxx` → E.164 `+989xxxxxxxxx`; validate `^\+989\d{9}$` (A-02) |
| Name | Unicode NFC; Arabic `ي`→`ی`, `ك`→`ک`; trim; collapse whitespace; remove bidi control chars (U+202A–202E, U+2066–2069, U+200E/F); keep ZWNJ; length 2–40 graphemes; must contain ≥ 2 letters; reject URLs/emails/digits-only |
| Reward text (admin) | NFC, same bidi-control stripping, length limits per field |

## 6. Dates and times

- Stored in UTC (`timestamptz`). Displayed in `Asia/Tehran` (A-06).
- Participant UI rarely shows dates; admin shows Jalali calendar via `Intl.DateTimeFormat('fa-IR-u-ca-persian', { timeZone: 'Asia/Tehran' })` with numeral policy applied.
- Exports include ISO 8601 UTC plus a Jalali local column.

## 7. Accessibility with RTL

- Screen-reader labels in Persian (`aria-label` from locale).
- Focus order follows visual RTL order.
- Do not convey state by color only; pair with icon/text [SPEC §28].
- Respect `prefers-reduced-motion` for non-essential UI motion.
