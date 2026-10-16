# 0020. v1 scope, preview modes, and performance target

- **Status:** Accepted
- **Date:** 2026-10-16

## Decision
- **All features in the PRD are equally required for v1.** There is no priority cut. The implementer still follows the dependency-ordered milestones in the PRD.
- **Preview modes:** classified terrain types (colours); raw field views (elevation, temperature, moisture, water distance, any intermediate field); toggleable overlays (resources, vegetation, spawn zones, POI markers, chunk grid). Hillshade is not required.
- **Performance:** dial changes should feel near-instant. The visible area at normal zoom refreshes in **≤ 200 ms**, and a full refresh at max zoom-out completes in **a few seconds (≤ 5 s target)**.
- **Out of scope for v1** (assumed; the author did not object): player edits / persisting world changes; wiring mine POIs to cave configs (the game's job); a node-graph UI; image/heightmap export (a screenshot of the preview is fine as a nice-to-have).
