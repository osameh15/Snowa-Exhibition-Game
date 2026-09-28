# Performance Budgets

Source: SPEC §27. Reference device: low-end/mid Android (tier A in [device matrix](../08-quality/02-device-browser-matrix.md)) on congested 4G (≈ 1.6 Mbps down, 150 ms RTT).

## 1. Participant web budgets

| Metric | Budget |
|---|---|
| App shell (HTML + JS + CSS + font subset + logo), compressed | ≤ 250 KB |
| JS for auth + lobby (compressed) | ≤ 170 KB (Vue + router + Pinia + app code; no UI mega-library) |
| Persian font | 1 family, 2 weights max, WOFF2 subset, ≤ 60 KB each |
| LCP (landing, cold, reference network) | ≤ 2.5 s |
| INP | ≤ 200 ms |
| CLS | ≤ 0.05 |
| Lobby usable (warm SW cache) | ≤ 1 s |
| Lobby thumbnails | ≤ 40 KB each (AVIF/WebP), lazy |

## 2. Game budgets

| Item | Budget |
|---|---|
| Phaser runtime chunk | ≤ 350 KB compressed (custom build excluding unused modules: physics engines not needed, etc.) — loaded once, cached |
| Game module code | ≤ 60 KB compressed per game |
| Critical asset group per game | ≤ 1.2 MB total (textures + audio) |
| Deferred group per game | ≤ 1.5 MB |
| Time from game tap to "ready" (cold, reference network) | ≤ 4 s; warm ≤ 1 s |
| Texture atlas max size | 2048 × 2048 (older iOS GPU limits) |
| GPU texture memory per game | ≤ 64 MB decoded |
| Frame rate | target 60 FPS; p5 ≥ 45 FPS on tier A mid devices; degrade VFX before dropping frames |
| Input-to-visual response | ≤ 1 frame + touch latency; no network in gameplay loop |
| JS heap after 3 games × 2 plays | growth ≤ 20 MB vs first play |

## 3. Image and audio guidance

| Asset | Format | Notes |
|---|---|---|
| UI icons | SVG (inline sprite) | Mirrorable for RTL where directional |
| Hero product renders | AVIF with WebP fallback, responsive `srcset` (1x/2x) | Largest ≤ 150 KB |
| Game sprites | PNG/WebP packed into atlases (TexturePacker or free equivalent), trimmed | WebP lossless where alpha needed; verify Safari support baseline (iOS 14+) |
| Particles/VFX | small grayscale textures tinted at runtime | |
| Audio SFX | one audio sprite per game, AAC (.m4a) + Opus/WebM | ≤ 150 KB per sprite; mono 44.1/48 kHz |
| Music (if any) | one short loop ≤ 300 KB, optional, deferred | SPEC §25 prefers SFX over music |
| Video backgrounds | avoided [SPEC §27] | |

Source art is kept separate from runtime exports; the asset pipeline (Phase 1 task) produces hashed, compressed outputs and a manifest per game.

## 4. Admin / display budgets

| Metric | Budget |
|---|---|
| Dashboard memory over 8 h | growth ≤ 10 % |
| Chart series | bounded windows (≤ 120 points) |
| Display animation | 60 FPS on booth mini-PC; CSS transforms only |
| Reorder animations | ≥ 2 s between reorders |
