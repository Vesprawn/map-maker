# 0013. Layers: stacked noise, climate fields, and terrain bands

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
By "layers" the author means all of: stacked noise that shapes land, biomes, terrain height bands, and separate data maps.

## Decision
A map is built from named scalar **fields**:
1. **Elevation**: a stack of noise layers (e.g. Continents, Mountains, Hills, Detail), each with dials (Scale, Strength, Roughness, plus Ridged/Billow style in Advanced) and a blend mode (add, multiply, max, min, mask-by-field).
2. **Climate**: **Temperature** (noise, optionally cooling with elevation, dial) and **Moisture** (noise, optionally increasing near water, dial). These are independent of height.
3. **Water** and **erosion**, per 0012 and 0010.
4. **Classification** into terrain types, per 0014. Terrain bands (sand → grass → rock → snow) and biomes (desert/forest/tundra) are both expressed as rules over elevation, temperature, moisture, and water distance.
5. **Data maps**, per 0016.

## Consequences
- The standard surface pipeline is a template built from generic ops. A cave map uses a different template (e.g. noise + threshold + cellular smoothing → floor/wall/ore) built from the same op library (0005).
- Noise primitives: seeded 2D simplex/OpenSimplex2 or Perlin, fBm, ridged, billow, domain warp.
