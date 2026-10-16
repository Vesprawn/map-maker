# 0016. Data maps: resources, vegetation, spawn zones, points of interest

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
Besides terrain type, the game needs ore/resource locations, vegetation density, spawn zones, and points of interest (villages, ruins, mine entrances).

## Decision
- **Resources, vegetation, spawn zones** are per-cell layers defined in the editor. Each one is noise × rules over existing fields/terrain types (e.g. "iron: in rock and mountain types, noise > 0.7"; "trees: density = moisture × forest-type mask"). Outputs are either a density value or a discrete id per cell (resource type id, spawn zone id). Like terrain types, their entries have stable ids.
- **Points of interest** are sparse and placed with a **deterministic region grid**. The world is divided into regions (e.g. 256×256 cells). Each region hashes (seed, region coords, poi type) to decide whether it has a POI and where it sits (jittered), then checks placement rules against the fields at that point (allowed terrain types, min distance to water, etc.). POI types are editor-defined with stable ids. Mine entrances are a POI type, and the game derives the cave map seed from (world seed, POI position) (0005).

## Consequences
- POI queries return a list per chunk (POIs whose position falls inside the chunk). Computing them only needs field values at the candidate points, so no large aprons.
- Min-spacing between POIs is guaranteed by the grid and jitter, not by global sampling.
