# Open Questions, Assumptions and Pending Decisions

Product decisions (owner: business/product) are kept separate from architecture decisions (ADRs, owner: engineering). "Blocks Phase 1" means Phase 1 implementation cannot finish (or cannot start, where stated) without an answer; non-blocking items have a safe default that is configurable or isolated behind an adapter.

## 1. Open questions

| ID | Question | Why it matters | Options | Recommended default | Owner | Blocks Phase 1 |
|---|---|---|---|---|---|---|
| OQ-01 | Approve backend technology? | Everything server-side | Node+TS+Fastify / NestJS / Go / Laravel / .NET | Node.js 22 + TS + Fastify ([ADR-002](../11-decisions/ADR-002-backend-technology.md)) | Engineering lead + sponsor | **Yes — blocks start** |
| OQ-02 | Hosting provider, region, managed PostgreSQL availability, TLS/CDN/WAF, data residency | Deployment, latency, reachability, backups | Domestic cloud / international cloud / on-prem | Provider with managed PostgreSQL + PITR, in the participants' country (A-05) | Snowa IT + DevOps | **Yes — staging needed by end of M1** |
| OQ-03 | OTP/SMS vendor, sender line, template (Persian + WebOTP line), limits, cost, test mode | Onboarding critical path | Vendor A/B/…, single vs dual | One primary vendor, optional secondary; platform-generated codes ([ADR-005](../11-decisions/ADR-005-otp-provider-boundary.md)) | Snowa + product | No (fake sender) — needed for staging real-SMS tests in M1 |
| OQ-04 | External Snowa API contract: URL, auth, fields (name?), score semantics (best vs attempt), delivery mode, idempotency support, response/error model, rate limits, sandbox, correction semantics | Adapter implementation, duplicates risk | per SPEC §17 | Send best score after each accepted attempt; request idempotency key support | Snowa IT | No (outbox skeleton + fake receiver); blocks Phase 6 |
| OQ-05 | Expected scale: peak concurrent users, total participants, displays, event days/hours | Load targets, sizing | — | A-01 values | Snowa events | No; blocks final load test (M7) |
| OQ-06 | Public leaderboard identity | Privacy, disambiguation | `NAME` / `NAME_INITIAL` / `NAME_MASKED_PHONE` / `MASKED_PHONE` | `NAME` (SPEC §3.1) | Product + legal | No (config); needed before public display use |
| OQ-07 | Numeral glyph policy (Persian vs Latin digits) per context | UI consistency | all Persian / Latin for numbers / mixed | Latin for scores/timers/ranks/phones/codes, Persian in prose ([Localization §4](../01-architecture/11-localization-and-rtl.md#4-numerals-oq-07--product-decision-required)) | UI design + brand | No (config) |
| OQ-08 | Raffle weighting: does each ticket add an entry? | Draw fairness & terms | ticket-weighted / uniform / per-draw choice | Per-draw choice, default ticket-weighted | Product + legal | No (Phase 5) |
| OQ-09 | Participant profile (name) editing allowed? History effect? | Scope, moderation | no edit / edit once / free edit | No edit in v1; admin moderation only | Product | No |
| OQ-10 | Can a previous winner win again in a later draw? | Prize distribution | allow / exclude per event / per draw | Per-draw option, default exclude | Operations + legal | No (Phase 5) |
| OQ-11 | Bilingual admin (English locale)? | i18n effort | Persian only / Persian + English | Persian only | Operations | No (Phase 2) |
| OQ-12 | OTP parameters: length, TTL, cooldown, attempts | UX vs security | — | 5 digits, 120 s, 60 s, 5 tries | Product + security | No (config) |
| OQ-13 | Data retention periods (phones, attempts, payloads, logs, backups) and post-event handling | Privacy/legal | — | Proposals in [Data lifecycle](../03-data/04-auditing-retention-lifecycle.md#3-retention-recommendations-oq-13--businesslegal-confirmation-required) | Legal + Snowa | No; before production |
| OQ-14 | Consent/terms: explicit checkbox before play? Final Persian text? | Legal; onboarding UX | no checkbox / checkbox on name step | Supported via flag, off until confirmed | Legal + product | No; before production |
| OQ-15 | Attempt consumption & interruption policy (consumed at start; no resume after reload; pause budget 60 s; late window 10 min) | Fairness vs exploit; core protocol | consume at start / consume at submit / N free restarts | As documented in [lifecycle §4](../02-domain/03-game-session-attempt-lifecycle.md#4-attempt-consumption-policy-answers-spec-82) + operator bonus attempts | Product | **Yes — sign-off needed during M1** (default implemented, values configurable) |
| OQ-16 | Vision Hunt: time-boxed vs fixed rounds; round timeout | Scoring ceiling, UX | time-boxed 40 s / fixed N rounds | Time-boxed, 5 s round timeout | Game design | No (Phase 4) |
| OQ-17 | Vision Hunt combo multiplier (concept art shows combo) | Score scale | none / like Spin / custom | None (SPEC text has none) | Game design | No (Phase 4) |
| OQ-18 | Minimum engagement for base ticket (e.g., score > 0 or ≥ 1 action)? | Ticket farming by idle play | none / min actions / min score | None (SPEC: completion) + config `min_score_for_base_ticket = 0` | Product | No |
| OQ-19 | Can running score go below zero (Fridge −20, Vision −50)? | Scoring | floor 0 / allow negative | Floor at 0 | Game design | No (Phase 3) |
| OQ-20 | Fridge Rush mystery object semantics | Reward logic boundary | cosmetic / reward trigger | Cosmetic in v1 | Product | No (Phase 3) |
| OQ-21 | Admin authentication: local + TOTP or Snowa SSO? | Security, ops | local+TOTP / SSO | Local + TOTP | Snowa IT + security | No (Phase 2) |
| OQ-22 | Event dates, hours, timezone, overnight behavior | Config, reward windows, runbook | — | Asia/Tehran; PAUSED overnight | Snowa events | No; before production |
| OQ-23 | When an attempt is invalidated, what happens to its base ticket and granted rewards? | Fairness | keep / void / re-point | Void base ticket only if no other valid attempt; otherwise re-point; rewards reviewed manually | Product + legal | No (Phase 2) |
| OQ-24 | Display-name moderation (deny-list, pre-moderation for public display) | Brand risk | none / deny-list / manual | Deny-list + operator hide | Brand | No (Phase 2) |
| OQ-25 | Final Persian game titles and Persian web font (license, weights) | UI build | — | Working titles; licensed font TBD | Brand | No (placeholder font in dev); needed before M7 |
| OQ-26 | Actual reward definitions, inventory, codes, fulfillment process | Reward config & ops | — | — | Snowa marketing | No; before production |
| OQ-27 | Domains (participant/admin), DNS, certificates | Deployment, cookies | subdomains / paths | `play.` and `admin.` subdomains | Snowa IT | No; before staging |
| OQ-28 | How are draw winners contacted and prizes handed over? | Ops process, data access | booth announcement / SMS / call | Booth announcement + operator call using revealed phone | Operations | No (Phase 5) |
| OQ-29 | Raffle legal rules: eligibility restrictions (staff, minors), terms, witnesses | Legal validity of draws | — | Must be provided | Legal | No (Phase 5) |
| OQ-30 | Analytics tooling: first-party only or approved third party? | Privacy, hosting | first-party / self-hosted analytics / third party | First-party only | Product + legal | No |
| OQ-31 | Are manual corrections (invalidate/restore, ticket grant/void) permitted during the event, and by whom? | Governance | disabled / Admin / Super Admin only | Enabled with permissions in [Roles](../06-admin/02-roles-and-permissions.md) | Operations + legal | No (Phase 2) |

## 2. Assumptions

| ID | Assumption | Revisit when |
|---|---|---|
| A-01 | Scale: 10k participants/day, 500 concurrent, 20 results/s (load-test ×5) | OQ-05 answered |
| A-02 | Iranian mobile numbers only (`+989XXXXXXXXX`) | Scope expands |
| A-03 | One active event at a time | Multi-event needed |
| A-04 | Participant & admin apps served same-origin with their APIs via reverse proxy | OQ-27 |
| A-05 | Runtime dependencies must be self-hosted; foreign services may be unreachable | OQ-02 |
| A-06 | Times stored UTC, displayed Asia/Tehran; admin uses Jalali calendar | OQ-22 |
| A-07 | OTP 5 digits (concept art) | OQ-12 |
| A-08 | Game durations: 40 s / 50 s / 40 s per SPEC | Prototype tuning |
| A-09 | Draw weighting default = active tickets | OQ-08 |
| A-10 | Initial scoring/difficulty values from SPEC; calibrated in prototypes, stored as config versions | Prototype results |
| A-11 | Participant session: absolute expiry = event end + 1 day (max 30 days) | Product/security review |
| A-12 | Admin auth = local accounts + mandatory TOTP | OQ-21 |
| A-13 | Device matrix models are representative placeholders | Market data |

## 3. Recommended architecture decisions pending approval

ADR-002 … ADR-012 with status *Proposed* — see [ADR index](../11-decisions/README.md).
