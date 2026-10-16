# 0011. Compare variants side by side

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The author wants to judge whether a change is better.

## Decision
The editor supports a **split view**: two configs (A and B) rendered over the **same world coordinates**, with pan/zoom locked together. B starts as a copy of A, and edits apply to the selected side. Either side can be promoted to become the main config. Undo/redo is also provided.

## Consequences
- The renderer and chunk cache must handle two configs at once (cache keyed by config hash + chunk coords).
- Doubles generation cost while comparing; workers must keep up.
- Implies saving/loading named configs (needed for export anyway).
