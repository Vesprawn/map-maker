# Architecture Decision Records

Each file records one decision: the context, what we chose, and why.
Number files sequentially (`NNNN-short-title.md`). Don't edit an accepted ADR to change the decision. Write a new one that supersedes it and update the old one's status.

| # | Title | Status |
|---|-------|--------|
| [0001](0001-purpose-game-world-terrain.md) | Purpose: game world terrain generator | Accepted |
| [0002](0002-platform-typescript-web-app.md) | Platform: TypeScript browser web app, single user | Accepted |
| [0003](0003-output-seed-and-params.md) | Output: seed + parameters, not baked images | Accepted |
| [0004](0004-world-shape-infinite-chunked.md) | World shape: infinite, chunked | Accepted |
| [0005](0005-caves-are-separate-maps.md) | Caves/mines are separate maps made with the same tool | Accepted |
| [0006](0006-pluggable-operations-with-aprons.md) | Terrain features are pluggable operations, with a declared apron | Accepted |
| [0007](0007-layer-stack-ui-graph-core.md) | Editor: layer stack UI over a graph-capable core | Accepted |
| [0008](0008-game-is-2d-top-down.md) | Target game is 2D top-down | Accepted |
| [0009](0009-feature-dials-advanced-hidden.md) | Controls: plain feature dials, advanced params hidden | Accepted |
| [0010](0010-erosion-water-proximity.md) | Erosion: a single generation-time dial acting on land near water | Accepted |
| [0011](0011-side-by-side-compare.md) | Compare variants side by side | Accepted |
| [0012](0012-water-model.md) | Water model: ocean, decorative rivers, lakes, wetlands | Accepted |
| [0013](0013-layers-fields-and-biomes.md) | Layers: stacked noise, climate fields, and terrain bands | Accepted |
| [0014](0014-editable-terrain-types.md) | Terrain types are editable and the tool classifies every cell | Accepted |
| [0015](0015-shared-ts-package.md) | Game is TypeScript: the generator ships as a shared package | Accepted |
| [0016](0016-data-maps-and-pois.md) | Data maps: resources, vegetation, spawn zones, points of interest | Accepted |
| [0017](0017-cell-equals-tile.md) | One generated cell equals one game tile | Accepted |
| [0018](0018-exact-preview-limited-zoom.md) | Preview is always exact; zoom-out is limited | Accepted |
| [0019](0019-editor-stack-and-persistence.md) | Editor stack: React + Vite; configs as JSON files with autosave; string/number seeds | Accepted |
| [0020](0020-v1-scope.md) | v1 scope, preview modes, and performance target | Accepted |
