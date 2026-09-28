# ADR-012: Deployment Topology

Status: **Accepted** for staging and exhibition v1 (resolves OQ-02 for Phase 1 direction; hosting vendor TBD and non-blocking)

## Context
Temporary, high-visibility exhibition; small team; low budget; possible restrictions on foreign cloud/CDN services; need for predictable operations and fast rollback. Security and integrity must not be simplified.

## Options
| Option | Assessment |
|---|---|
| Kubernetes | Unjustified overhead |
| Docker on several VMs + managed PostgreSQL | More moving parts and cost than needed for v1 |
| PaaS | Availability in target region uncertain |
| Serverless | Poor fit for SSE and background jobs |
| **One VPS: reverse proxy + static Nuxt build + one Fastify process (systemd) + PostgreSQL** | Cheapest and simplest; portable to any Linux VPS vendor |

## Decision
- **Staging:** `https://snowa-games.osameh.dev`; Ubuntu Server 24.04 LTS; 1 vCPU / 2 GB RAM / 25–30 GB SSD/NVMe / ~2 GB swap; static IPv4; ≥ ~100 Mbps; prefer location close to Iranian users.
- **Exhibition production starting point:** 2 vCPU / 4 GB RAM / 40–50 GB SSD/NVMe, PostgreSQL on the same VPS; final size from load-test evidence.
- Nginx **or** Caddy, Let's Encrypt TLS, static SPA files, `/api` proxy with SSE streaming.
- Fastify supervised by `systemd` (no PM2, no Docker requirement), dedicated service user.
- Off-server encrypted backups (daily minimum; tighter RPO for event production).
- No Redis, Kubernetes, message broker, separate worker or separate frontend deployment in v1.
- Incremental scaling path: larger VPS → separate PostgreSQL → separate worker → multiple Fastify instances → optional Redis ([Deployment topology §5](../01-architecture/10-deployment-topology.md#5-incremental-scaling-path-future-options-not-v1-requirements)).

## Consequences
+ Very low cost and operational surface; easy rehearsal and rebuild.
− Single point of failure; deploys cause a brief (seconds) restart gap → mitigated by idempotent client retries, late-submission window, deploys outside peak, fast restore procedure.
− Vertical limits → scaling stages available without re-architecture.
