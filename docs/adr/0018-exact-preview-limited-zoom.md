# 0018. Preview is always exact; zoom-out is limited

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
At 1 cell = 1 tile (0017), wide views contain millions of cells. Approximate LOD previews would make the preview differ from what the game generates.

## Decision
The preview **always renders exact chunk output**, the same data the game gets. Zoom-out is capped so the visible area stays about **≤ 2048×2048 tiles** (exact cap tuned to hit performance targets). There is no approximate LOD.

## Consequences
- What you see is what the game gets, at every zoom level.
- To survey a large area, the author pans. A future "overview render" job (slow, exact, offscreen) could be added if needed.
- Chunk generation must be fast enough to fill about 1,000 chunks of 64² at max zoom-out in reasonable time across a worker pool.
