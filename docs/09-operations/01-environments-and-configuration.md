# Environments, Configuration and Secrets

## 1. Environments

| Env | Purpose | Data | Integrations | Access |
|---|---|---|---|---|
| `local` | Development | Seeded fixtures | Fake OTP, fake Snowa | Developers |
| `ci` | Automated tests | Ephemeral (Testcontainers) | Fakes | CI |
| `staging` | QA, device tests, load tests, rehearsal | Synthetic; production-like volume for load | Real SMS provider test mode or fake (flag); Snowa sandbox (TBD) or fake receiver | Team + stakeholders; banner "محیط آزمایشی" |
| `production` (event) | Live exhibition | Real | Real SMS, real Snowa | Restricted; admin origin allowlisted if possible |

Production and staging have separate databases, secrets, domains, SMS sender configurations and admin accounts. No production data is copied to staging (privacy); if needed, only anonymized.

## 2. Configuration layers

| Layer | Examples | Changed by | Requires deployment |
|---|---|---|---|
| Build-time | API base path, build hash, feature compile flags | CI | yes |
| Environment (env vars) | DB URL, SMS adapter selection, Snowa base URL, log level, allowed origins, fake-OTP flag (must be `false` in production, startup assertion) | DevOps | restart |
| Secrets | DB password, SMS API key, Snowa credentials, OTP pepper, session/IP hash keys, TOTP encryption key | DevOps via secret store | restart |
| Event policy (DB) | OTP parameters, pause budget, late window, public identity policy, numeral policy, consent flag, core ticket policy, rate-limit values | Super Admin (audited) | no |
| Live controls (DB) | game state, attempt limits, reward rules, displays | Admin/Operator | no |
| Game config versions (DB, immutable) | tuning params, bounds | Super Admin publish (between event days) | no, unless new code needed |
| Locale (`i18n-fa`) | Persian copy | Content owner via PR | yes (static) |

Startup validation: the API refuses to start if required config is missing/invalid, or if production runs with fake adapters.

## 3. Secrets management

- Stored in the provider's secret manager, or in an encrypted file (SOPS/age) deployed to hosts with root-only permissions if no manager exists (OQ-02).
- Never in git, images, client bundles or logs; CI secret scanning.
- Rotation: all production secrets generated fresh before the event; rotate SMS/Snowa credentials after the event.
- Peppers/HMAC keys: changing them invalidates OTP challenges / phone hashes — rotate only with a documented procedure.
