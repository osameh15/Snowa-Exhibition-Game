# Participant Lifecycle, Onboarding and OTP

Source: SPEC §4, §6, PD-02, PD-03, AC-001, AC-002. API: [Auth & profile API](../04-api/02-auth-and-profile-api.md). Security: [Authentication](../07-security/02-authentication-and-sessions.md).

## 1. Participant record lifecycle

```mermaid
stateDiagram-v2
  [*] --> VERIFIED_NO_NAME: first successful OTP (row created)
  VERIFIED_NO_NAME --> ACTIVE: display name saved
  ACTIVE --> BLOCKED: admin blocks (reason, audited)
  BLOCKED --> ACTIVE: admin unblocks (audited)
  ACTIVE --> ERASED: data erasure (retention/legal, Super Admin)
  VERIFIED_NO_NAME --> ERASED
  ERASED --> [*]
```

- A participant row is created **only after successful OTP verification** (no rows for unverified phones → no enumeration, no junk).
- Persisted `participants.status` has three values (`ACTIVE`, `BLOCKED`, `ERASED`). `VERIFIED_NO_NAME` is a **derived** state: `status = ACTIVE AND display_name IS NULL`.
- `BLOCKED`: cannot create game sessions; existing sessions revoked; excluded from public leaderboards and draw eligibility; data retained.
- `ERASED`: phone replaced by irreversible hash, name removed; attempts retained in anonymized form if retention policy requires (OQ-13).

## 2. Onboarding state machine

```mermaid
stateDiagram-v2
  [*] --> Anonymous
  Anonymous --> PhoneEntered: valid phone format
  PhoneEntered --> AwaitingOtp: challenge SENT
  PhoneEntered --> PhoneEntered: rate limited / send failed (retry after cooldown)
  AwaitingOtp --> AwaitingOtp: wrong code (attempts left)
  AwaitingOtp --> PhoneEntered: edit phone / challenge expired / locked
  AwaitingOtp --> Authenticated: code verified (session cookie set)
  Authenticated --> NameRequired: display_name is null
  Authenticated --> InLobby: display_name present
  NameRequired --> InLobby: name saved
  InLobby --> Anonymous: logout / session expired / revoked
```

Rule: the server rejects lobby and game endpoints with `403 PROFILE_INCOMPLETE` while `display_name` is null, so skipping the name screen client-side is impossible (AC-001).

## 3. OTP challenge state machine

```mermaid
stateDiagram-v2
  [*] --> CREATED: request accepted (rate limits passed)
  CREATED --> SENT: provider accepted
  CREATED --> SEND_FAILED: provider error/timeout
  SENT --> VERIFIED: correct code before expires_at
  SENT --> SENT: wrong code (attempts remain)
  SENT --> LOCKED: verify_attempts = max
  SENT --> EXPIRED: now > expires_at
  SENT --> SUPERSEDED: newer challenge for same phone
  SEND_FAILED --> [*]
  VERIFIED --> [*]
  LOCKED --> [*]
  EXPIRED --> [*]
  SUPERSEDED --> [*]
```

| Parameter (event config) | Recommended default | Notes |
|---|---|---|
| `otp.length` | 5 | Concept art shows 5 digits (CA-01); OQ-12 |
| `otp.ttl_seconds` | 120 | |
| `otp.resend_cooldown_seconds` | 60 | Concept shows ~45 s |
| `otp.max_verify_attempts` | 5 per challenge | then LOCKED |
| `otp.max_sends_per_phone` | 5 / 15 min, 10 / 24 h | |
| Code generation | `crypto.randomInt(0, 10^len)` zero-padded | |
| Code storage | `HMAC-SHA256(pepper, challenge_id ‖ code)`; plaintext never stored/logged | |
| Comparison | constant-time | |

Only the **latest** SENT challenge for a phone is verifiable (older → SUPERSEDED), so resend does not multiply guessing chances.

## 4. Sequence — first-time participant

```mermaid
sequenceDiagram
  autonumber
  actor V as Visitor
  participant C as Participant app
  participant API as API (identity)
  participant DB as PostgreSQL
  participant SMS as OtpSender adapter
  V->>C: Scan QR → https://play.domain/?src=booth-a
  C->>C: load app shell (cached if revisit) · record qr_entry (anon_id, src)
  V->>C: enter phone (Persian or Latin digits)
  C->>API: POST /api/v1/auth/otp/request {phone} + Idempotency-Key
  API->>API: normalize → +989XXXXXXXXX · validate · rate-limit (phone, IP class)
  API->>DB: supersede open challenges · INSERT otp_challenge(CREATED, code_hash, expires_at)
  API->>SMS: send(phone, code)
  SMS-->>API: accepted
  API->>DB: challenge → SENT · analytics otp_requested
  API-->>C: 200 {challengeId, expiresAt, resendAvailableAt, codeLength}
  Note over API,C: Same response shape whether or not the phone is known (no enumeration)
  SMS-->>V: SMS with code (+ WebOTP line)
  V->>C: enter/auto-fill code
  C->>API: POST /api/v1/auth/otp/verify {challengeId, code}
  API->>DB: lock challenge · check state/expiry/attempts · constant-time compare
  API->>DB: INSERT participant (VERIFIED_NO_NAME) ON CONFLICT(phone) DO NOTHING · INSERT participant_session
  API-->>C: 200 {participant:{id, displayName:null, profileComplete:false}} + Set-Cookie sx_ps (HttpOnly, Secure, SameSite=Lax)
  C->>C: route → /onboarding/name
  V->>C: enter name
  C->>API: PUT /api/v1/me/profile {displayName}
  API->>API: normalize + validate name
  API->>DB: UPDATE participant SET display_name WHERE display_name IS NULL · analytics profile_completed
  API-->>C: 200 {profileComplete:true}
  C->>API: GET /api/v1/lobby
  API-->>C: lobby (3 game cards, tickets 0/3)
  C->>V: Lobby with greeting
```

## 5. Sequence — returning participant

```mermaid
sequenceDiagram
  autonumber
  actor V as Visitor
  participant C as Participant app
  participant API as API
  participant DB as PostgreSQL
  alt valid session cookie still present
    C->>API: GET /api/v1/me
    API-->>C: 200 {profileComplete:true}
    C->>API: GET /api/v1/lobby
  else no/expired session
    V->>C: enter phone
    C->>API: POST /auth/otp/request
    API-->>C: 200 {challengeId,…}
    V->>C: enter code
    C->>API: POST /auth/otp/verify
    API->>DB: participant exists (ACTIVE, has name) → INSERT session
    API-->>C: 200 {profileComplete:true} + Set-Cookie
    C->>API: GET /api/v1/lobby
  end
  API->>DB: read current game_settings, progress, tickets (never cached per participant)
  API-->>C: lobby with best scores, attempts used/allowed, ranks, tickets, reward history link
  C->>V: Lobby (no name prompt — AC-002)
```

A `BLOCKED` participant receives `403 PARTICIPANT_BLOCKED` at verify time with a neutral Persian message.

## 6. Display name policy

| Topic | Rule |
|---|---|
| Set | Once, when null |
| Edit | Not supported in v1 (OQ-09). Admin can correct/hide a name (audited) for moderation |
| Validation | See [Localization §5](../01-architecture/11-localization-and-rtl.md#5-input-normalization-client-and-server) |
| Uniqueness | Not unique (display data only) |
| Public use | Per leaderboard identity policy (OQ-06) |
| Moderation | Optional deny-list filter; admin `hide_display_name` flag replaces public name with a neutral placeholder (OQ-24) |

## 7. Consent (OQ-14)

If explicit consent is required, the name step (first time) shows a required checkbox. The server stores `consent_records(participant_id, terms_version, accepted_at, ip_hash)`. The lobby endpoint enforces a current consent version when `event.policies.require_consent = true`. Disabled by default until confirmed.
