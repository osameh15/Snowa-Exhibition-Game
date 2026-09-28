# Entity Definitions

Suggested physical design. Names are final unless changed by an approved ADR; types are PostgreSQL. `PK` primary key, `FK` foreign key, `UQ` unique, `IX` index, `CK` check.

## 1. Identity

### `participants`
| Column | Type | Constraints / notes |
|---|---|---|
| id | uuid | PK (UUIDv7) |
| phone_e164 | text | UQ, CK `~ '^\+989[0-9]{9}$'` (A-02); nullable only after erasure |
| phone_hash | bytea | UQ; HMAC(pepper, phone) — kept after erasure for dedupe |
| display_name | text | nullable until onboarding; CK length 2..60 chars (validated 2..40 graphemes in app) |
| display_name_hidden | boolean | default false (moderation) |
| status | text | CK in (`ACTIVE`,`BLOCKED`,`ERASED`); default `ACTIVE` |
| verified_at | timestamptz | first OTP success |
| last_seen_at | timestamptz | updated at most every 60 s per participant |
| created_at / updated_at / version | | |

IX: `(lower(display_name))` trigram (admin search; `pg_trgm`), `(created_at)`.

### `otp_challenges`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| phone_e164 | text | IX `(phone_e164, created_at DESC)` |
| code_hash | bytea | HMAC(pepper, id ‖ code) |
| status | text | CK (`CREATED`,`SENT`,`SEND_FAILED`,`VERIFIED`,`LOCKED`,`EXPIRED`,`SUPERSEDED`) |
| verify_attempts | smallint | default 0, CK ≤ 20 |
| expires_at | timestamptz | |
| resend_available_at | timestamptz | |
| ip_prefix | text | coarse network (e.g., /24) for rate-limit analytics |
| provider_message_id | text | nullable |
| participant_id | uuid | FK, set on VERIFIED |
| created_at / updated_at | | |

Retention: 30 days (OQ-13) — used for abuse investigation only.

### `participant_sessions`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| token_hash | bytea | UQ; SHA-256 of the 256-bit cookie token (token itself never stored) |
| participant_id | uuid | FK, IX |
| created_at, last_used_at, expires_at | timestamptz | absolute expiry (A-11: event end + 1 day, max 30 days) |
| revoked_at | timestamptz | nullable |
| user_agent_hash | text | |

### `consent_records` (only if OQ-14 requires)
`(id PK, participant_id FK, terms_version text, accepted_at, ip_hash)` UQ `(participant_id, terms_version)`.

## 2. Catalog

### `events`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| slug | text | UQ |
| name_fa | text | |
| status | text | CK (`DRAFT`,`LIVE`,`PAUSED`,`CLOSED`) |
| brand_key | text | brand profile key (e.g., `snowa`) — see [Reuse & branding](../01-architecture/14-reuse-and-branding.md) |
| timezone | text | default `Asia/Tehran` |
| starts_at / ends_at | timestamptz | OQ-22 |
| policies | jsonb | validated schema: `core_ticket` (incl. `min_score_for_base_ticket`, OQ-18), `pause_budget_ms`, `submit_grace_ms`, `late_submission_window_ms`, `public_identity`, `numeral_policy`, `require_consent`, `otp`, `draw_defaults` |
| version, created_at, updated_at | | |

Partial UQ: at most one `LIVE` event: `UNIQUE (status) WHERE status = 'LIVE'`.

### `games`
`id smallint PK` (1 `spin-perfect`, 2 `fridge-rush`, 3 `vision-hunt`), `slug text UQ`, `external_ref text` (Snowa game id mapping, OQ-04), `title_fa`, `sort_order`. Seeded by migration; not admin-creatable in v1.

### `game_settings`
| Column | Type | Notes |
|---|---|---|
| event_id, game_id | | PK `(event_id, game_id)`; FKs |
| state | text | CK (`ENABLED`,`DISABLED`,`EMERGENCY_STOPPED`) |
| attempt_limit | integer | CK `1..100`; default 1 [SPEC PD-05] |
| active_config_version_id | uuid | FK `game_config_versions` |
| state_reason | text | required for EMERGENCY_STOPPED |
| version, updated_at, updated_by | | optimistic concurrency |

### `game_config_versions`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| game_id | smallint | FK |
| version | text | UQ `(game_id, version)`, semver-like `1.0.0` |
| runtime_min_version | text | minimum client runtime build accepted |
| params | jsonb | gameplay/tuning parameters (schema per game in game-core) |
| bounds | jsonb | computed & reviewed: `max_score`, `max_actions`, `min_action_interval_ms`, plausibility thresholds |
| checksum | text | SHA-256 of canonical `params` — client echoes it |
| status | text | CK (`DRAFT`,`PUBLISHED`,`RETIRED`) — PUBLISHED rows immutable (trigger) |
| created_at, published_at, published_by | | |

## 3. Play

### `game_sessions`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK (UUIDv4 random — unguessable) |
| event_id, game_id, participant_id | | FKs; IX `(participant_id, game_id, state)` |
| config_version_id | uuid | FK |
| seed | bytea | 16 bytes CSPRNG |
| state | text | CK (`ISSUED`,`STARTED`,`SUBMITTED`,`EXPIRED`,`CANCELLED`) |
| issued_at, start_by, started_at, deadline_at, late_deadline_at, submitted_at, ended_at | timestamptz | |
| create_idempotency_key | text | |
| client_info | jsonb | permitted device metadata: UA family, OS, viewport class, DPR, runtime build [SPEC §12.2] |

Partial UQ (INV-02): `UNIQUE (participant_id, game_id) WHERE state IN ('ISSUED','STARTED')`.
IX for sweeper: `(state, start_by) WHERE state='ISSUED'`, `(state, late_deadline_at) WHERE state='STARTED'`.

### `attempts`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| seq | bigint | identity, UQ — global acceptance order & tie-break |
| session_id | uuid | FK, **UQ** (INV-05) |
| event_id, game_id, participant_id | | FKs |
| attempt_number | integer | UQ `(event_id, participant_id, game_id, attempt_number)` (INV-04) |
| status | text | CK (`IN_PROGRESS`,`ACCEPTED`,`ACCEPTED_FLAGGED`,`REJECTED`,`ABANDONED`,`INVALIDATED`) |
| claimed_score | integer | nullable |
| attempt_score | integer | nullable; CK ≥ 0; set only for ACCEPTED* |
| reject_reason | text | code, nullable |
| active_ms, paused_ms, server_elapsed_ms | integer | |
| started_at, submitted_at, accepted_at | timestamptz | `accepted_at` via `clock_timestamp()` |
| invalidated_at, invalidated_by, invalidation_reason, status_before_invalidation | | |
| created_at, updated_at | | |

IX: `(event_id, game_id, status, attempt_score DESC)` (reports), `(participant_id, game_id)`, `(accepted_at)`.

### `attempt_payloads`
`attempt_id PK/FK`, `payload jsonb` (raw client submission: action log, claimed stats; ≤ 128 KB), `payload_sha256`, `recomputed jsonb` (server replay stats), `evaluation_trace jsonb` (reward rolls), `received_at`. Separate table keeps `attempts` narrow and allows earlier purge of raw logs.

### `attempt_flags`
`(attempt_id FK, code text, detail jsonb, created_at)` PK `(attempt_id, code)`. Codes: `LATE_SUBMISSION`, `REACTION_TOO_FAST`, `INPUT_RATE_HIGH`, `PERFECT_RATE_HIGH`, `TIMING_JITTER_LOW`, `DURING_EMERGENCY_STOP`, `REWARD_EVAL_DEFERRED`, `DUPLICATE_MISMATCH`, `NEAR_SCORE_CEILING`.

### `participant_game_progress`
| Column | Type | Notes |
|---|---|---|
| event_id, participant_id, game_id | | PK |
| attempts_used | integer | CK ≥ 0 |
| bonus_attempts | integer | default 0, CK 0..10 |
| best_score | integer | nullable |
| best_attempt_id | uuid | FK attempts, nullable |
| best_attempt_seq | bigint | nullable |
| best_achieved_at | timestamptz | nullable |
| first_valid_completion_at | timestamptz | nullable |
| base_ticket_id | uuid | FK, nullable |
| ranked | boolean | generated/maintained: `best_score IS NOT NULL AND NOT excluded_from_ranking AND participant ACTIVE` |
| excluded_from_ranking | boolean | default false (admin) |
| updated_at, version | | |

CK: `(best_score IS NULL) = (best_attempt_id IS NULL)`.
IX leaderboard: `(event_id, game_id, best_score DESC, best_achieved_at ASC, best_attempt_seq ASC) WHERE ranked`.

## 4. Tickets and rewards

### `raffle_tickets`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK |
| event_id, participant_id | | FKs; IX `(event_id, participant_id) WHERE status='ACTIVE'` |
| source_type | text | CK (`BASE`,`REWARD`,`ADMIN`) |
| game_id | smallint | required for BASE |
| source_attempt_id | uuid | FK, required for BASE |
| reward_grant_id | uuid | FK, required for REWARD |
| admin_action_id | uuid | required for ADMIN |
| ordinal | smallint | 1..N within a grant/action |
| status | text | CK (`ACTIVE`,`VOID`) |
| voided_at, voided_by, void_reason | | |
| created_at | | |

Partial UQ (INV-07): `UNIQUE (event_id, participant_id, game_id) WHERE source_type='BASE'`.
UQ: `(reward_grant_id, ordinal) WHERE source_type='REWARD'`; `(admin_action_id, ordinal) WHERE source_type='ADMIN'`.

### `reward_definitions`
`id PK`, `event_id FK`, `type` CK (`DISCOUNT_CODE`,`EXTRA_TICKETS`,`PHYSICAL_PRIZE`,`BENEFIT`,`SPECIAL`), `fulfillment_mode` CK (`CODE_POOL`,`SHARED_CODE`,`AUTO`,`CLAIM`,`INSTRUCTIONS`), `title_fa`, `description_fa`, `terms_fa`, `shared_code` (nullable), `ticket_quantity` (EXTRA_TICKETS), `valid_until`, `status`, `version`, audit columns.

### `reward_rules`
`id PK`, `definition_id FK`, `trigger` CK, `game_ids smallint[]` (null=any), `min_attempt_score`, `min_best_score`, `probability_bp` CK 0..10000, `active_from`, `active_until`, `total_limit` (nullable), `granted_count` CK `granted_count >= 0`, `per_participant_limit` CK ≥ 1, `exclusive_group`, `priority`, `status` CK (`DRAFT`,`ACTIVE`,`PAUSED`,`ARCHIVED`), `unlimited_confirmed boolean`, `version`, audit columns.
CK: `total_limit IS NOT NULL OR unlimited_confirmed`. (Cap overrun prevented by conditional update, not by CK, because lowering `total_limit` below `granted_count` is allowed.)

### `reward_participant_counters`
PK `(rule_id, participant_id)`, `granted_count` CK ≥ 0.

### `reward_codes`
`id PK`, `definition_id FK`, `code text`, UQ `(definition_id, code)`, `status` CK (`AVAILABLE`,`ASSIGNED`,`VOID`), `grant_id` UQ nullable (INV-08), `uploaded_batch_id`, `created_at`, `assigned_at`.
IX: `(definition_id, id) WHERE status='AVAILABLE'` (fast SKIP LOCKED allocation).

### `reward_grants`
`id PK`, `event_id`, `participant_id FK`, `definition_id FK`, `rule_id FK`, `rule_snapshot jsonb`, `source_attempt_id FK`, `code_id FK` (nullable, UQ), `status` CK (`GRANTED`,`CLAIM_PENDING`,`CLAIM_CONTACTED`,`FULFILLED`,`REDEEMED`,`REVOKED`), `fulfillment_note`, `fulfilled_by`, `fulfilled_at`, `created_at`, `version`.
UQ: `(rule_id, source_attempt_id)`.

## 5. Raffle

### `draws`
`id PK`, `event_id FK`, `name_fa`, `criteria jsonb`, `weighting` CK (`TICKETS`,`UNIFORM`), `winner_count` CK ≥ 1, `allow_fewer boolean`, `state` CK (`DRAFT`,`CANCELLED`,`COMPLETED`,`PRESENTING`,`REVEALED`,`VOIDED`), `preview_count`, `eligible_count`, `total_weight bigint`, `snapshot_hash bytea`, `seed bytea`, `algorithm_version text`, `app_version`, `config_snapshot jsonb`, `executed_by`, `executed_at`, `execute_idempotency_key` UQ, `revealed_count` CK `0..winner_count`, `replaces_draw_id FK`, `void_reason`, `voided_by`, `voided_at`, `created_by`, `created_at`, `version`.
Trigger: after `COMPLETED`, columns other than `state`, `revealed_count`, void fields are immutable.

### `draw_entries`
PK `(draw_id, ordinal)`; `participant_id FK`; `weight integer CK > 0`; UQ `(draw_id, participant_id)`. Insert-only.

### `draw_winners`
PK `(draw_id, position)`; `participant_id`; UQ `(draw_id, participant_id)` (INV-11); `entry_ordinal`; `weight`. Insert-only.

## 6. Integration

### `external_deliveries`
| Column | Type | Notes |
|---|---|---|
| id | uuid | PK; idempotency key toward Snowa |
| attempt_id | uuid | FK, UQ `(attempt_id, kind)` |
| kind | text | `RESULT` (future: `CORRECTION`) |
| payload | jsonb | canonical model snapshot |
| status | text | CK (`PENDING`,`IN_FLIGHT`,`RETRY_SCHEDULED`,`DELIVERED`,`FAILED`,`SUPERSEDED`,`RESOLVED_MANUALLY`) |
| attempt_count | integer | |
| next_attempt_at | timestamptz | IX `(status, next_attempt_at)` |
| locked_until | timestamptz | lease for IN_FLIGHT recovery |
| last_error_category / last_error_detail | text | redacted |
| delivered_at, created_at, updated_at | | |

### `external_delivery_attempts`
`(id PK, delivery_id FK, attempted_at, duration_ms, http_status, error_category, response_excerpt (≤ 1 KB, redacted))`.

## 7. Admin and operations

- `admin_users`: `id`, `username` UQ, `display_name`, `password_hash` (argon2id), `totp_secret_enc` (encrypted with KMS/app key), `role` CK (`OPERATOR`,`ADMIN`,`SUPER_ADMIN`), `status`, `failed_login_count`, `locked_until`, `last_login_at`, audit columns.
- `admin_sessions`: like participant sessions + `idle_expires_at`, `ip_prefix`, `mfa_verified_at`.
- `display_devices`: `id`, `event_id`, `name`, `token_hash` UQ, `mode jsonb`, `last_seen_at`, `revoked_at`.
- `audit_log`: see [Audit model](../02-domain/08-audit-model.md).
- `idempotency_keys`: PK `(scope, key)` where scope = `participant:<id>` / `admin:<id>` / `anon:<ip-prefix>`; `request_hash`, `status` (`IN_PROGRESS`,`COMPLETED`), `response_code`, `response_body jsonb`, `created_at`; TTL 24 h (purged by a background job).
- `rate_limit_buckets`: PK `(scope, key, window_start)`, `count integer`; fixed-window counters updated with `INSERT … ON CONFLICT DO UPDATE SET count = count + 1 RETURNING count`; purged hourly. Replaced by Redis if load tests require (ADR-004).
- `analytics_events`: `id bigint`, `event text`, `occurred_at`, `received_at`, `anon_id uuid`, `participant_id uuid null`, `game_id`, `props jsonb`; IX `(event, occurred_at)`; optional daily partitioning.
