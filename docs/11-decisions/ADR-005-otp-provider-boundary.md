# ADR-005: OTP Provider Boundary

Status: **Proposed**. Vendor is **not** selected (OQ-03) — SPEC §30.2 forbids inventing it.

## Context
SMS delivery is on the critical onboarding path; provider availability, template rules, WebOTP formatting, sender IDs and rate limits vary by vendor. Iranian SMS providers typically expose simple HTTPS APIs with template/verification endpoints; candidates are evaluated by the business, not assumed here.

## Decision
- Domain code depends on an `OtpSender` port: `send(phoneE164, code, locale) → {accepted, providerMessageId} | {error: TRANSIENT|PERMANENT, category}`.
- The platform generates, hashes and verifies codes itself (provider only delivers text). Avoid provider-side "verify" APIs so verification rules (attempts, TTL, supersede) stay under platform control and switching vendors is trivial.
- Adapters: `FakeOtpSender` (dev/CI/staging-optional, blocked in production), `<Vendor>OtpSender` (TBD), optional secondary vendor with automatic failover when the primary returns transient errors for > N% in 2 min (config).
- Message template (Persian) includes the WebOTP line `@play.<domain> #<code>` if the vendor allows free-text or an approved template supports it.
- Timeouts 5 s; no retries inside the request beyond one fast retry on connect error (to avoid double SMS); the participant can resend after cooldown.

## Vendor evaluation checklist
Delivery rate/latency to all Iranian operators, template approval process and lead time, free-text vs template, sender line type, throughput limits, sandbox/test mode, pricing, status/delivery reports API, uptime history, support during event hours.

## Consequences
+ Vendor swap without domain changes; testable without SMS.
− Two-vendor failover doubles integration/testing effort (optional).
