# ADR-001: Frontend Architecture

Status: **Accepted** — Nuxt 4 + Vue 3 + TypeScript + Phaser 3 + PWA baseline; single-app delivery accepted in the final Phase 0 correction (supersedes the earlier separate-admin-app refinement).

## Context
SPEC §30.3 and the approved baseline select Nuxt 4, Vue 3, TypeScript, Phaser 3 for gameplay, PWA, normal web UI outside Phaser. The exhibition deployment must be simple and inexpensive: one frontend build, no always-running SSR server.

## Options considered
| Topic | Options | Choice |
|---|---|---|
| Rendering | SSR / hybrid / SPA (`ssr: false`) / prerendered shell | **Static SPA/PWA** — authenticated, QR-launched app needs no SEO or server rendering; served as static files by the reverse proxy |
| App split | separate participant/admin apps / one app with route groups | **One app**: participant routes, `/admin`, `/display`, each as lazily loaded route chunks. Security is enforced by the API ([Admin architecture](../01-architecture/06-admin-architecture.md)), not by separate deployments |
| State | Pinia / composables only | **Pinia** for session/lobby; server state re-fetched, not cached long-term |
| Styling | Tailwind / UnoCSS / plain CSS + tokens | **CSS custom-property design tokens (from the brand profile) + logical properties**; utility framework optional if it enforces logical properties |
| UI kit | Vuetify/Quasar / custom | **Custom minimal components** |
| PWA | `@vite-pwa/nuxt` (Workbox) / hand-written SW | **@vite-pwa/nuxt** with custom runtime caching rules; SW scope excludes `/admin` and `/display` caching of API data |
| i18n | `@nuxtjs/i18n` / tiny custom | **@nuxtjs/i18n**, single `fa` locale, brand copy overrides |

## Decision
One Nuxt 4 static SPA/PWA (`apps/web`) with three route groups (participant `/`, `/admin`, `/display`), sharing `contracts`, `i18n-fa`, `ui`, `game-core` and the active brand profile. Phaser and each game module load only when a game route is opened. No Nuxt SSR server in v1; SSR may be reconsidered only if a concrete need appears.

## Consequences
+ One build and one static deployment; perfect caching; no frontend runtime process.
+ Route-level code splitting keeps admin/display code off participant phones.
− Admin shares the participant origin → mitigated by strict CSP, cookie path scoping, step-up TOTP and server-side authorization; a separate admin subdomain remains a future hardening option.
