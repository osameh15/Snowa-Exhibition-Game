# ADR-001: Frontend Architecture

Status: **Accepted (baseline)** — Nuxt 4 + Vue 3 + TypeScript + Phaser 3 + PWA is the approved frontend baseline. Refinements below are **Proposed**.

## Context
SPEC §30.3 and the approved baseline select Nuxt 4, Vue 3, TypeScript, Phaser 3 for gameplay, PWA, normal web UI outside Phaser. Open implementation choices: rendering mode, app split, state management, styling.

## Options considered
| Topic | Options | Choice |
|---|---|---|
| Rendering | SSR / hybrid / SPA (`ssr: false`) / prerendered shell | **SPA with static shell** — authenticated, QR-launched app gains nothing from SSR; SPA removes a Node rendering tier and caches perfectly |
| App split | one app with `/admin` / separate admin app | **Separate `apps/admin`** — smaller participant bundle, separate origin & cookies, IP allowlist possible |
| State | Pinia / composables only | **Pinia** (Nuxt standard) for session/lobby; server state re-fetched, not cached long-term |
| Styling | Tailwind / UnoCSS / plain CSS + tokens | **CSS custom-property design tokens + logical properties**; utility framework optional if it enforces logical properties (e.g., Tailwind v4 `ms-/me-` utilities) |
| UI kit | Vuetify/Quasar / custom | **Custom minimal components** — heavy kits inflate the bundle and fight RTL/brand styling |
| PWA | `@vite-pwa/nuxt` (Workbox) / hand-written SW | **@vite-pwa/nuxt** with custom runtime caching rules ([Caching](../10-performance/02-caching-loading-network.md)) |
| i18n | `@nuxtjs/i18n` / tiny custom | **@nuxtjs/i18n** with single `fa` locale, lazy messages |

## Decision
Nuxt 4 SPA for participant and admin apps in a pnpm monorepo, sharing `contracts`, `i18n-fa`, `ui`, `game-core`.

## Consequences
+ Static hosting, simple scaling, strong caching, small participant bundle.
+ Admin isolation improves security posture.
− No SSR SEO (irrelevant for this product).
− Two Nuxt apps to build/deploy (same pipeline).
