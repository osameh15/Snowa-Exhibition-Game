# Privacy and Data Protection

Legal requirements (Iranian or other jurisdiction) are **not** asserted here; every retention period, consent requirement and sharing basis below requires business/legal confirmation (OQ-13, OQ-14).

## 1. Personal data inventory

| Data | Class | Purpose | Visible to | Shared externally |
|---|---|---|---|---|
| Phone number | P1 | Authentication, prize contact, Snowa integration | Participant (own, masked), Admin/Super Admin (reveal, audited) | Snowa API (as agreed), SMS provider (for OTP) |
| Display name | P2 | Greeting, leaderboard, draw presentation | Public per identity policy; admins | Snowa only if contract requires (OQ-04) |
| IP address | P2 | Rate limiting, abuse investigation | Stored as keyed hash + coarse prefix; raw only in short-lived proxy logs | No |
| User agent / device class | P2 | Compatibility, anti-cheat context | Admin (attempt detail) | No |
| Scores, attempts, tickets, rewards | P3 | Game function, ranking, draws | Participant (own), admins, public (scores with name) | Scores to Snowa |
| Analytics events | P3 | Product/ops metrics | Admin reports | No third parties (default) |

## 2. Principles applied

- Minimization: only phone + name are mandatory inputs [SPEC §4.2]; no email, birthdate, location.
- Public projection never includes phone (AC-017); identity policy configurable (OQ-06).
- Admin access by role; full phone reveal is an audited, per-participant action.
- Logs: phone masked (`+98912•••6789`), names omitted, OTP never logged, tokens never logged; request bodies not logged for auth/result endpoints.
- Analytics: no phone/name; pseudonymous ids only; no third-party analytics without approval [SPEC §23].
- Exports: minimum columns, phone opt-in by permission, audited, one-time download.
- External sharing boundary: the Snowa payload contains only fields in the approved contract; changing it requires a documented decision.
- SMS provider receives phone + message text only.
- Consent/terms: versioned consent record if required (OQ-14); privacy notice in Persian linked from phone screen.
- Retention & erasure: see [Data lifecycle](../03-data/04-auditing-retention-lifecycle.md).
- Participant rights (access/erasure requests): handled by Super Admin via erase function; process owner TBD by business.

## 3. Hosting location

Data residency (where the DB and backups live) must be confirmed together with hosting (OQ-02). Backups must be encrypted and access-controlled with the same rigor as the primary database.
