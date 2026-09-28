# ADR-013: Reuse by Configuration, Not Multi-Tenancy

Status: **Accepted**

## Context
The platform is a temporary Snowa exhibition product but may later be reused for other exhibitions, companies or brands. Building multi-tenant SaaS now would add cost and risk without a requirement.

## Options
| Option | Assessment |
|---|---|
| Hard-code Snowa throughout | Fastest now; reuse requires rewrites |
| **Configurable brand profile + clean module/adapter boundaries; one brand per deployment** | Small upfront discipline; reuse by new deployment + configuration |
| Full multi-tenant SaaS (tenant ids everywhere, provisioning, billing, isolation) | Large scope, unjustified now |

## Decision
Separate brand identity, theme tokens, copy, game titles/assets, product imagery, reward definitions, event configuration and external integration adapter from platform code, as specified in [Reuse & branding](../01-architecture/14-reuse-and-branding.md). Keep explicit `Event`, brand profile and game catalog concepts. Do not build tenant provisioning, billing, isolation, SaaS administration or tenant switching.

## Consequences
+ A new brand needs configuration, assets, copy and possibly an adapter — not changes to OTP, sessions, scoring, leaderboard, raffle, rewards, RBAC, audit or delivery framework.
− Serving several brands simultaneously requires separate deployments (acceptable).
