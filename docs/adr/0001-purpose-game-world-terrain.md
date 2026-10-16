# 0001. Purpose: game world terrain generator

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
"Map noise generator" could mean noise-texture tooling, art maps, dungeon layouts, or game terrain. Scope and every later choice depend on which.

## Decision
The tool generates **terrain for a game world**: elevation and derived terrain features, driven by procedural noise.

## Alternatives considered
- Stylised fantasy art maps: rejected; looking good matters less than having usable terrain data.
- Raw noise-texture workbench: too general, with no terrain semantics.
- Tile/dungeon generation: a different problem domain.

## Consequences
- Output must be meaningful as terrain (height, and possibly biomes), not just a pretty image.
- Labels, typography, and print-quality cartography are out of scope.
