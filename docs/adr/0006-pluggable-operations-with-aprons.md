# 0006. Terrain features are pluggable operations, with a declared apron

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The author wants to experiment with erosion, rivers, layers, etc., and doesn't yet know which algorithms they want. Some algorithms are pointwise (noise). Others need neighbouring data (simulated erosion), which conflicts with independent infinite chunks (0004).

## Decision
Every terrain feature is an **operation** implementing a common interface: typed inputs, typed outputs, a parameter schema, and an **apron** (the number of border cells it needs beyond the requested area). Multiple algorithms can fill the same role (e.g. several erosion ops) and are swappable. To compute a chunk, the evaluator works backwards through the pipeline, growing the requested area by each op's apron, then crops the result.

Ops that need a coarse global structure (e.g. river networks) work on a **deterministic coarse region grid** keyed by region coordinates. They never use whole-world passes.

## Alternatives considered
- Hard-coding one algorithm per feature: blocks experimentation.
- Finite-map-only global simulation: violates 0004.

## Consequences
- Large aprons make chunks expensive. The UI should show each op's cost/apron.
- Neighbourhood ops are only approximately seam-free: results near the apron edge differ from a whole-world sim. Seams must be tested for explicitly.
- New algorithms can be added without changing the core or the config format.
