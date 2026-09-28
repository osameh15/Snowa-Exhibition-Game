# Real-Time Architecture

Decision record: [ADR-003](../11-decisions/ADR-003-realtime-transport.md). Event contracts: [Real-time events](../04-api/07-realtime-events.md).

## 1. Principle

Real-time messages are **hints**; PostgreSQL is authoritative [SPEC §19.2]. Every real-time consumer has a REST snapshot endpoint and MUST be able to recover purely by re-fetching.

## 2. Transport choice

| Channel | Consumers | Transport | Reason |
|---|---|---|---|
| Admin stream | Admin dashboard, live scores, raffle console | SSE | One-way server→client; commands go over REST |
| Display stream | Public booth screens | SSE | One-way; auto-reconnect built into `EventSource` |
| Leaderboard stream | Participant leaderboard screen (only while visible) | SSE, closed on `visibilitychange: hidden` | Avoid persistent connections on phones |
| Participant lobby/result | Participants | REST fetch on demand | Rank is shown on result response and lobby load; no stream needed |
| Fallback | Any SSE consumer | Poll snapshot every 5–10 s after 3 failed reconnects | Proxies/networks that break streaming |

WebSocket is not selected: no client→server streaming requirement exists, and SSE works over plain HTTP/1.1 and HTTP/2 with standard proxies, cookies and auth, with less operational surface.

## 3. Fan-out across API replicas

```mermaid
sequenceDiagram
  participant H as api replica A (handles submit)
  participant PG as PostgreSQL
  participant B as api replica B
  participant C1 as Admin (SSE on A)
  participant C2 as Display (SSE on B)
  H->>PG: COMMIT result tx
  H->>PG: NOTIFY rt, '{"type":"attempt_accepted","game":"spin-perfect",...}'
  PG-->>H: notification
  PG-->>B: notification
  H->>H: assign event id, append to ring buffer
  B->>B: assign event id, append to ring buffer
  H-->>C1: event
  B-->>C2: event
```

- Payloads on NOTIFY are small (< 1 KB; PostgreSQL limit 8000 bytes) and contain ids, not PII.
- Each replica keeps a **5-minute ring buffer** per channel to serve `Last-Event-ID` resumption. Event ids are `<epochMs>-<replicaSeq>`; if a client reconnects to a different replica or the id is older than the buffer, the server sends a `resync` event and the client re-fetches the snapshot.
- `leaderboard_changed` is **coalesced**: at most one per game per second per replica.
- Heartbeat comment (`: ping`) every 15 s keeps proxies and mobile NAT alive; clients treat 35 s without data as disconnected.

## 4. Connection budget

| Consumer | Expected connections (A-01) | Notes |
|---|---|---|
| Admin/operator | ≤ 20 | |
| Displays | ≤ 10 | |
| Participant leaderboard viewers | ≤ 500 concurrent peak | Only while leaderboard screen visible |

Node.js handles thousands of idle SSE connections per replica; the constraint is proxy configuration (`proxy_buffering off`, read timeout ≥ 60 s, HTTP/2 to avoid the 6-connections-per-origin limit of HTTP/1.1).

## 5. Behavior under failure

| Failure | Consumer sees | Recovery |
|---|---|---|
| Stream drop | "reconnecting" badge after 5 s | `EventSource` retry (server sends `retry: 3000`); snapshot re-fetch on reconnect |
| API replica restart | Drop, reconnect to another replica | `resync` → snapshot |
| NOTIFY listener lost DB connection | Replica stops receiving hints | Listener reconnects; on reconnect emits `resync` to its clients |
| DB down | Snapshot fetch fails | Displays keep last data with explicit **stale** banner; never present stale data as live |
| Draw reveal during disconnect | Display misses reveal events | Reveal state (`revealed_count`) is persisted; on reconnect the display renders the persisted state |
