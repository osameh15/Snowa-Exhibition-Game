# Threat Model

Method: STRIDE per trust boundary ([Trust boundaries](../01-architecture/09-trust-boundaries.md)). Assets ranked by business impact.

## 1. Assets

| Asset | Why it matters |
|---|---|
| Leaderboard integrity (best scores, ranks) | Public fairness, rank-based prizes |
| Raffle integrity (tickets, eligibility, winners) | Prizes, legal/brand risk |
| Reward inventory (codes, physical prizes) | Direct cost |
| Participant PII (phones, names) | Privacy, reputation |
| Admin accounts | Control of everything above |
| OTP/SMS budget | Direct cost (SMS pumping) |
| Availability on event day | Campaign success |
| External API credentials | Snowa system access |

## 2. Adversaries

| Adversary | Capability | Motivation |
|---|---|---|
| Curious participant | DevTools, replaying requests | Top rank, more tickets |
| Scripted cheater | Reverse-engineers client, writes bots | Prizes |
| Multi-account farmer | Several SIM cards/phones | More tickets/rewards |
| SMS pumper / bot | Automated OTP requests | Cost/abuse |
| Malicious insider / careless operator | Admin access | Favoritism, mistakes |
| Offensive-name troll | Types abusive display name | Embarrass brand on public display |
| Network attacker | Venue Wi-Fi | Session theft |
| DoS | Traffic floods | Disruption |

## 3. Threats and mitigations

| ID | STRIDE | Threat | Mitigation | Residual |
|---|---|---|---|---|
| T-01 | Tampering | Submit arbitrary score | Server replay of action log; claimed score ignored; bounds | Low |
| T-02 | Tampering | Forge plausible action log (bot) | Hard physical limits, plausibility flags, top-rank review, invalidation | **Medium (accepted)** |
| T-03 | Repudiation/Tampering | Replay a previous good submission into a new session | Payload bound to session id + seed (different layouts/zones), config checksum; one result per session | Low |
| T-04 | Elevation | Play more attempts than allowed | Server-side limit under row lock; attempt consumed at start | Low |
| T-05 | Tampering | Reload to reroll a bad start | Attempt consumed at start; no resume; new seed per session | Low |
| T-06 | Tampering | Duplicate submissions for extra tickets/rewards | Session state, unique constraints, idempotency | Low |
| T-07 | Spoofing | OTP brute force | 5 digits, 5 tries/challenge, only latest challenge valid, lockouts, per-phone send limits | Low |
| T-08 | DoS/Cost | SMS pumping / OTP flood | Per-phone and per-IP-class limits, Iranian mobile prefixes only, global SMS budget circuit breaker, optional challenge (captcha) toggle | Low–Medium |
| T-09 | Spoofing | Session theft on shared Wi-Fi | HTTPS/HSTS, HttpOnly Secure cookies, no tokens in URLs/localStorage | Low |
| T-10 | Tampering | CSRF on participant/admin actions | SameSite=Lax, custom header, Origin check | Low |
| T-11 | Information disclosure | Phone numbers leaked via public APIs/displays | Public DTOs without phone fields; masking; tests (AC-017) | Low |
| T-12 | Information disclosure | Enumerate registered phones | Uniform OTP request responses | Low |
| T-13 | Information disclosure | Access other participants' sessions/results | Ownership checks, UUIDv4 ids, 404 on foreign ids | Low |
| T-14 | Elevation | Admin account compromise | TOTP, lockout, IP allowlist option, least privilege, audit, alerts on sensitive actions | Low–Medium |
| T-15 | Tampering | Insider manipulates draw | Server selection, recorded seed/snapshot, execution-request audit committed separately, no re-roll, void requires Super Admin, hash-chained audit | Low |
| T-16 | Tampering | Insider grants self rewards/tickets | Permissions, reason, audit, reports of manual actions | Low |
| T-17 | DoS | Traffic flood on event day | Proxy rate limits, connection limits, static shell caching, autoscaling if provider supports; WAF/CDN if available | Medium |
| T-18 | Tampering | Reward inventory race (overspend) | Atomic conditional updates, SKIP LOCKED codes | Low |
| T-19 | Spoofing | Display token leaked (someone shows fake board) | Tokens revocable, display only renders server data; token scoped read-only | Low |
| T-20 | Tampering | Offensive/bidi-spoofed names on public display | Name normalization, bidi-control stripping, deny-list, hide-name moderation | Low–Medium |
| T-21 | Information disclosure | Secrets in logs/client | Secret scanning in CI, log redaction, no secrets in bundles | Low |
| T-22 | Tampering | Multi-account farming (many SIMs) | One account per phone; cannot fully prevent; analytics on device fingerprints *not* used (privacy) — accepted business risk | **Medium (accepted)** |
| T-23 | Supply chain | Malicious npm dependency | Lockfile, pinned versions, `npm audit`/OSV scanning, minimal deps, SRI not applicable (self-hosted) | Low–Medium |

## 4. Accepted risks (require business acknowledgment)

- T-02 scripted plausible scores (browser games cannot be cheat-proof) [SPEC §21.1].
- T-22 multiple phone numbers per person.
- Ambiguous external API duplicates if Snowa lacks idempotency (R-05).
