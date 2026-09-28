# Source Analysis Notes

Record of how the product source was analyzed, what it contains, and where it is ambiguous or internally inconsistent. The source file itself (`docs/source/…Spec_v1.0.docx`) is not modified.

## 1. Coverage

| Source part | Analyzed | Notes |
|---|---|---|
| §0–§32 body text | Yes | All sections, including acceptance criteria (§29) and open items (§32) |
| All tables | Yes | Decisions PD-01..PD-12, roles, scoring tables, filters, admin controls, data model, AC-001..AC-022 |
| Appendix A (Persian copy starter set) | Yes | Used as initial locale key list |
| Appendix B (traceability) | Yes | IDs reused in [Traceability matrix](04-requirement-traceability-matrix.md) |
| Appendix C (5 concept boards) | Yes | Overall journey, Spin Perfect, Fridge Rush, Vision Hunt, Admin dashboard |

## 2. Concept-art observations

The five boards embedded in the SPEC are also available as files (byte-identical to the embedded images): [overall journey + admin](../images/UIUX.png), [Spin Perfect](../images/first-game-ui.png), [Fridge Rush](../images/second-game-ui.png), [Vision Hunt](../images/third-game-ui.png), [admin dashboard](../images/admin-panel.png).

Concept art is explicitly "visual direction, not pixel-perfect UI" [SPEC Appendix C]. Where art differs from text, **text wins**. Observations relevant to engineering:

| # | Observation | Treatment |
|---|---|---|
| CA-01 | OTP screen shows a **5-digit** code with a resend countdown (~0:45). | Default OTP length 5, configurable (OQ-12). |
| CA-02 | Name screen in Fridge Rush board shows a **terms acceptance checkbox** ("با شروع بازی، قوانین و شرایط را می‌پذیرم"). | Consent requirement is open [SPEC §32]; architecture supports a versioned consent record (OQ-14). |
| CA-03 | Leaderboard mock shows **masked phone numbers** (e.g., `0912***6789`) instead of names; admin leaderboard shows names. | SPEC §3.1 recommends names; masked phone only if stakeholders require. Public identity policy is OQ-06. |
| CA-04 | Leaderboard mock has tabs "all / friends / me". | "Friends" is not in SPEC text; excluded. "Me" maps to "own rank highlight". |
| CA-05 | Numbers use **Latin digits** for scores, ranks, phone and timers; admin mock mixes Persian and Latin digits and shows a **Jalali date**. | Numeral policy is OQ-07; Jalali display for admin is recommended (A-06). |
| CA-06 | Vision Hunt HUD shows a **pause button**, a "3/5" target counter and a **combo x3**; result shows "13/15 items" and a **01:12** play time. | Conflicts with the 40 s duration baseline and SPEC defines no combo for Vision Hunt. Round model and combo are OQ-16/OQ-17; text baseline (≈40 s) is used. |
| CA-07 | Spin Perfect result shows "PERFECT +120" in-game (= 100 × 1.2 combo multiplier) and "COMBO x4". | Consistent with SPEC §9.3 multiplier table. |
| CA-08 | Result screens show attempt score, NEW BEST badge, previous best, rank, ticket card, optional extra reward (discount code). | Matches SPEC §13.5 order. |
| CA-09 | Admin mock: live online players, total plays, total participants, avg play time, per-game toggles, attempt steppers, core reward "Raffle Ticket" (dropdown), extra reward buttons, live draw panel (all players / score above X / rank range / specific game / all three), winner count, recent activity, participation donut. | Directly informs [Admin IA](../06-admin/01-information-architecture.md). The core reward appears as a dropdown in the mock; per SPEC §13.2 it is fixed and not editable in normal operation. |
| CA-10 | Profile screen shows "my rewards", "game rules", "FAQ", "logout". | Rules/FAQ are static content pages (optional, Persian). Logout is supported. |
| CA-11 | Lobby bottom navigation: home, leaderboard, rewards, profile. | Adopted as recommended navigation (final design decides). |

## 3. Textual ambiguities found

| # | Ambiguity | Resolution in this package |
|---|---|---|
| AMB-01 | "Valid completion" is not defined (does score 0 count? a rejected result?). | Defined as attempt status `ACCEPTED` or `ACCEPTED_FLAGGED`; score may be 0. Minimum-engagement rule for ticket is OQ-18. |
| AMB-02 | When exactly an attempt is consumed [SPEC §8.2]. | Consumed at session **start** (after assets load and the participant presses start); ISSUED-but-unstarted sessions never consume. See [Game lifecycle](../02-domain/03-game-session-attempt-lifecycle.md). |
| AMB-03 | Refresh/background policy [SPEC §22.2]. | Pause budget + hard deadline + no gameplay resume after page reload; policy is versioned config. OQ-15 for product sign-off. |
| AMB-04 | Whether tickets are weighted entries in a draw. | SPEC §13.3 "additional raffle ticket adds extra entries" implies weighting. Recommended: weighted by active ticket count; confirmation required (OQ-08). |
| AMB-05 | Which score is sent externally [SPEC §17.3]. | Default: current best score; canonical model carries both attempt and best score (OQ-04). |
| AMB-06 | Fridge Rush negative scores (wrong placement −20). | Recommended: running score floored at 0 (tuning parameter, OQ-19). |
| AMB-07 | Spin Perfect: "Great" combo effect "maintain or limited". | Tuning parameter; default "maintain". |
| AMB-08 | Vision Hunt round timeout and round count. | Time-boxed session; per-round timeout skip (tuning, OQ-16). |
| AMB-09 | Previous winners in later draws [SPEC §15.3]. | Per-draw option; recommended default "exclude previous winners of this event" (OQ-10). |
| AMB-10 | "Online/active players" definition [SPEC §16.1]. | Defined technically: distinct participants with an authenticated API request in the last 2 minutes, plus a separate "currently playing" count (sessions STARTED and not past deadline). |
| AMB-11 | Emergency stop in-progress policy [SPEC §16.2]. | Blocks new sessions and starts; already STARTED sessions may still submit (data preserved; admin may invalidate). |
| AMB-12 | Fridge Rush mystery bonus object [SPEC §10.6]. | Cosmetic in v1; rewards decided only by server rules at result time (OQ-20). |
| AMB-13 | Admin language: Persian/RTL expected [SPEC §5] vs English technical labels in admin mock. | Admin UI Persian/RTL; bilingual admin is OQ-11. |

## 4. Items SPEC explicitly leaves open (§32)

All are carried into [Open questions](../12-planning/04-open-questions.md): OTP provider, external API contract, hosting, event dates, expected scale, prize legal rules, product imagery, final Persian titles, font, admin language, profile edit policy, retention, consent copy, reward inventory, public leaderboard naming.
