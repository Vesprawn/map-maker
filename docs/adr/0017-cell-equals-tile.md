# 0017. One generated cell equals one game tile

- **Status:** Accepted
- **Date:** 2026-10-16

## Decision
Generation runs at tile resolution: **1 cell = 1 tile**. Chunks are square, with a configurable size (default **64×64**) stored in the config. World coordinates are integer tile coordinates.

## Consequences
- The chunk API returns typed arrays of length size² (row-major), one per field/layer.
- Noise "Scale" dials are expressed in tiles (e.g. "feature size ≈ 400 tiles"), which is intuitive.
- Float precision: noise inputs are tile coords × frequency. To stay precise far from the origin, sample coordinates are offset per region/chunk with integer math where possible. Document a supported range (e.g. ±1,000,000 tiles).
