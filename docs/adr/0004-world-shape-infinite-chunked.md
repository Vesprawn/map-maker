# 0004. World shape: infinite, chunked

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
Options were a single finite map, a finite island, an infinite world, or a wrapping world.

## Decision
The world is **infinite and generated in chunks on demand**. Any chunk at integer coordinates (cx, cy) can be computed independently from the config alone.

## Alternatives considered
- Finite map / island: simpler, but doesn't match the game's needs.
- Wrapping/tileable: not required.

## Consequences
- Generation must be a **pure function of (config, world coordinates)**, with no global passes over the whole map. Global falloff masks are impossible. Erosion and river simulation that need neighbouring data become hard: they require chunk borders/aprons or another approach.
- Chunk edges must be seamless: sample in world space, never per-chunk local space.
- Very large coordinates risk float precision loss; the domain needs a strategy for that.
- The previewer needs pan/zoom and a chunk cache.
