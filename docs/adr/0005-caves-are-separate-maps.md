# 0005. Caves/mines are separate maps made with the same tool

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
Caves were first considered as a feature inside the surface world, which would force a 3D/voxel data model. The real need is that players create mines on the surface, and entering one loads a different map.

## Decision
Caves are **not** part of the surface map. A cave/underground level is its **own map config**, built in the same editor from the same operation library. The tool supports multiple map configs, such as a "surface" map and a "cave" map.

## Alternatives considered
- 3D density/voxel world: far heavier, and unnecessary for a 2D top-down game (see 0008).
- A cave overlay layer on the surface heightmap: couples two different levels together.

## Consequences
- The data model stays 2D scalar fields.
- The operation library must be general enough to produce cave-style output (floor/wall layouts, e.g. thresholded noise or cellular automata), not only landforms.
- Linking a mine entrance on the surface to a cave map (seed derivation, entrance placement) is **out of scope for v1** and is the game's responsibility. A likely convention is cave seed = hash(world seed, entrance coords).
