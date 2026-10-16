# 0009. Controls: plain feature dials, advanced params hidden

- **Status:** Accepted
- **Date:** 2026-10-16
- **Refines:** 0006 (UI only; the operation architecture stays the same)

## Context
The author thinks in terms of features and intensity ("turn erosion up"), not algorithm choice or simulation parameters.

## Decision
Each layer/feature shows a **small set of plain-language dials** (e.g. Strength, Amount, Size, Roughness) with sensible ranges and defaults. An **"Advanced" toggle** per layer reveals the raw algorithm parameters, and the algorithm choice where more than one exists. Dials map to raw parameters through a documented function in each operation.

## Alternatives considered
- Dials only: blocks deeper tinkering later.
- Everything exposed: too noisy for routine use.

## Consequences
- Every operation defines `dials` (a user-facing schema) and a `dialsToParams()` mapping, alongside its raw param schema.
- The config stores the dial values and any advanced overrides, so the map can be reproduced exactly.
- Algorithm swapping (0006) stays available but is tucked away.
