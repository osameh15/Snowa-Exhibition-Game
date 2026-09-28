# ADR-012: Deployment Topology

Status: **Proposed** (provider/region pending OQ-02)

## Context
Short-lived, high-visibility event; small team; unknown hosting; possible restrictions on foreign cloud/CDN services; need for predictable operations and fast rollback.

## Options
| Option | Assessment |
|---|---|
| Kubernetes | Powerful; operational overhead unjustified at this scale |
| **Docker containers on 2 VMs + managed/self-hosted PostgreSQL** | Simple, portable to any provider (domestic or international), rolling restarts via Compose/scripts |
| PaaS (e.g., app platforms) | Easiest if available in the chosen region; availability uncertain |
| Serverless | Poor fit for SSE and workers; vendor lock-in |

## Decision
Containerized `api` (×2), `worker` (×1), reverse proxy (Caddy or Nginx) with static bundles, PostgreSQL (managed preferred). Provider-agnostic scripts/Compose files; infrastructure documented as code where the provider allows. All runtime dependencies self-hosted.

## Consequences
+ Portable across providers; minimal moving parts; easy to rehearse.
− Manual-ish scaling (add a VM/replica) — acceptable given known event dates.
− If the provider lacks managed PostgreSQL, the team owns replication/backups (document and drill).
