# Environments, Configuration and Secrets

## 1. Environments

| Env | Purpose | Host | Data | Integrations | Access |
|---|---|---|---|---|---|
| `local` | Development | Developer machine (Node.js 22 + local PostgreSQL 16) | Seeded fixtures | Fake OTP, fake Snowa receiver | Developers |
| `ci` | Automated tests | CI runner (PostgreSQL service/Testcontainers in CI only) | Ephemeral | Fakes | CI |
| `staging` | QA, device tests, load tests, rehearsal | **`https://snowa-games.osameh.dev`** — one VPS (Ubuntu 24.04, 1 vCPU / 2 GB / 25–30 GB, ~2 GB swap), PostgreSQL on the same VPS | Synthetic | Real SMS vendor test mode or fake (flag); Snowa sandbox (TBD) or fake receiver | Team + stakeholders; persistent banner "محیط آزمایشی"; optional proxy basic auth / IP allowlist |
| `production` (event) | Live exhibition | One VPS (starting 2 vCPU / 4 GB / 40–50 GB), PostgreSQL on the same VPS; domain TBD (OQ-27) | Real | Real SMS, real Snowa | Restricted |

No managed database is required for staging or production v1 ([ADR-004](../11-decisions/ADR-004-persistence.md)). Production and staging never share databases, secrets, SMS sender configuration or admin accounts. Production data is never copied to staging (privacy).

## 2. Configuration layers

| Layer | Examples | Changed by | Requires deployment |
|---|---|---|---|
| Build-time (web) | API base path, build hash, active brand key (`BRAND=snowa`) | CI | yes |
| Environment (server) | `DATABASE_URL`, `PUBLIC_ORIGIN`, `JOBS_ENABLED` (true in v1), `OTP_ADAPTER`, `EXTERNAL_RESULT_ADAPTER`, Snowa base URL, log level, `FAKE_OTP` (must be false in production — startup assertion) | Operator | restart |
| Secrets | DB password, SMS API key, Snowa credentials, OTP pepper, session/IP hash keys, TOTP encryption key | Operator | restart |
| Event policy (DB) | OTP parameters, pause budget, late window, public identity, numerals, consent flag, core ticket policy, rate limits | Super Admin (audited) | no |
| Live controls (DB) | game state, attempt limits, reward rules, displays | Admin/Operator | no |
| Game config versions (DB, immutable) | tuning params, bounds | Super Admin publish (between event days) | no, unless new code needed |
| Brand profile (`brands/<key>/`) | logo, tokens, copy, game titles/assets | Content owner via PR | yes (static) |

Startup validation: Fastify refuses to start if required configuration is missing/invalid, if production runs with fake adapters, or if `PUBLIC_ORIGIN` is not HTTPS.

## 3. Secrets management

- Stored outside source control in a root-owned environment file referenced by the systemd unit (`EnvironmentFile=/etc/snowa-games/server.env`, mode `0600`, owner `root`, read by systemd before dropping privileges), or systemd credentials (`LoadCredential=`). Encrypted copies of the files are kept in the team's password manager/vault.
- Never in git, client bundles, logs or backups of the repository; CI secret scanning.
- Separate secrets per environment; generated fresh before the event; SMS/Snowa credentials rotated after the event.
- Peppers/HMAC keys: rotating them invalidates OTP challenges / phone hashes — documented procedure only.
