# ADR-006: Game Runtime

Status: **Accepted** (Phaser 3 baseline; shared deterministic `game-core` structure confirmed together with ADR-010).

## Context
Three 2.5D-looking, touch-driven, short games; 60 FPS on mid-range Android; lazy loading per game; gameplay must be replayable server-side for validation.

## Options
| Option | Notes |
|---|---|
| **Phaser 3** (baseline) | Mature 2D engine, WebGL + Canvas fallback, input, tweens, audio, scale manager, texture atlases; large but tree-shakable via custom build |
| PixiJS + custom game loop | Smaller renderer, more code to write (input, audio, scenes) |
| Phaser 4 | Newer renderer; ecosystem/maturity risk for a fixed-date event |
| DOM/CSS only | Insufficient for smooth VFX-rich gameplay |

## Decision
Phaser 3 for rendering/input/audio, with **all scoring logic in `packages/game-core`** (pure TS, integer math, seeded PRNG, no Phaser/DOM imports), used by Phaser scenes and by the server validator. Vue owns the shell, HUD text and all non-gameplay screens. One Phaser game instance per game route; destroyed on exit. Physics engines excluded from the build (not needed).

## Consequences
+ Proven engine; team familiarity per baseline.
+ Deterministic core makes server validation and golden tests possible.
− Discipline required: no `Math.random`/`Date.now`/floats in scoring paths (lint rules in game-core).
− Phaser bundle (~300+ KB compressed) → loaded lazily and cached once.
