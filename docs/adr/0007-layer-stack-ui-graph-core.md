# 0007. Editor: layer stack UI over a graph-capable core

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The options were a layer stack (ordered list), a node graph (World Machine/Gaea style), or a fixed pipeline.

## Decision
The **UI is an ordered layer stack**: add, remove, reorder, toggle, and swap an algorithm per layer, and edit params. The **config and evaluator are a DAG of operations**. A layer stack is a linear DAG, plus references to named earlier outputs where needed (e.g. using a moisture field as a mask). A node-graph editor can be added later without changing the config format.

## Alternatives considered
- Node graph in v1: roughly doubles UI work for a single user.
- Fixed pipeline: too rigid for experimentation.

## Consequences
- The config schema stores nodes + edges (or an equivalent), even though v1 UI only creates near-linear graphs.
- Need blend modes (add, multiply, max, min, lerp-by-mask) as ops or per-layer settings.
