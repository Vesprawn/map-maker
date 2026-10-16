# 0008. Target game is 2D top-down

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
What "height", "cave map", and "terrain" mean depends on the game's perspective.

## Decision
The consuming game is **2D top-down**. Generated fields (height, moisture, etc.) are inputs the game maps onto tile/terrain types. A 3D mesh preview is not needed.

## Consequences
- The preview is 2D: colour-mapped fields, with optional hillshade for readability.
- The final output of a map is a set of named 2D fields, often classified into discrete terrain/tile types.
- Cave maps are 2D floor/wall layouts.
