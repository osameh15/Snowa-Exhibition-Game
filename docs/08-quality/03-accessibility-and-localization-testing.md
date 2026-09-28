# Accessibility and Localization Testing

## 1. Localization (Persian/RTL)

| Test | Method | Pass criteria |
|---|---|---|
| No English UI text (AC-003) | E2E DOM scan for `[A-Za-z]{2,}` outside allowlist (brand, codes, ids) on every participant route and result state | 0 violations |
| Missing keys | Build-time check `i18n-fa` vs used keys | build fails on missing |
| RTL layout | Visual regression screenshots (Playwright) at 360×640, 390×844, 430×932, 768×1024 | approved baselines |
| Logical CSS | stylelint rule forbids physical properties in shared UI | lint green |
| Mixed content | Snapshot tests for phone, OTP, score, timer, rank, discount code, names (Latin & Persian) inside Persian sentences | correct visual order |
| Digit input | Phone/OTP accept Persian, Arabic-Indic and Latin digits; paste with spaces/dashes | normalized server-side |
| Numeral policy | Switching config changes glyphs without component changes | pass |
| Long names | 40-grapheme names, ZWNJ, mixed scripts truncate safely on leaderboard/result/display | no overflow |
| Dates (admin) | Jalali + Tehran time rendering around midnight/Nowruz boundaries | correct |
| Copy review | Brand/content owner signs off all Persian strings | sign-off recorded |

## 2. Accessibility

| Area | Check |
|---|---|
| Contrast | WCAG 2.1 AA for text; score/timer/action states high contrast [SPEC §28] |
| Color independence | Every state distinguishable in grayscale screenshots |
| Touch targets | ≥ 48 dp UI; game-specific minimums (Fridge zones ≥ 64 dp, Vision objects ≥ 48 dp) measured at 360 px |
| Flashing | Automated frame analysis or manual review: ≤ 3 flashes/s |
| Reduced motion | `prefers-reduced-motion` disables shake/extra particles |
| Screen reader | Auth, lobby, result, leaderboard usable with TalkBack/VoiceOver in Persian labels |
| Zoom | Browser zoom 200 % on non-game screens without loss of function |
| Audio | Games fully playable muted; sound toggle visible |
| Recovery | "Back to lobby" path on every result/error [SPEC §28] |
| Automated scan | axe-core in E2E on shell routes | no serious/critical issues |
