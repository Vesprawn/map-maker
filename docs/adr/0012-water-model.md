# 0012. Water model: ocean, decorative rivers, lakes, wetlands

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The world needs ocean, rivers, lakes, and swamps/wetlands, and every one must be computable per chunk (0004). The author chose **decorative** rivers: they should look like rivers but don't need physically correct flow.

## Decision
- **Ocean:** cells with elevation below a global **Sea level** dial.
- **Rivers (decorative):** pointwise. A river exists where a low-frequency, domain-warped noise field is near zero (|n| < width). Rivers carve their channel into elevation so they sit in valleys. They are suppressed above a max elevation and blend into the ocean at the coast. Dials: **Amount** (frequency), **Width**, **Winding** (warp strength), **Depth** (carve). No global flow graph.
- **Lakes:** pointwise. Low-frequency "basin" noise selects lake regions, and inside them cells below a local lake-water level become water. The lake surface level is a smooth function of position, so a lake has one flat level and no stepped or tilted water. Dials: **Amount**, **Size**.
- **Wetlands:** not a water body. They are a **terrain classification** (0014) driven by low elevation, high moisture, and proximity to water, with patchy noise. Dial: **Wetness**.

The pipeline outputs a `water` field (none/ocean/river/lake) and a `waterDistance` field consumed by erosion (0010) and classification.

## Alternatives considered
- Realistic flow-routed rivers: needs a coarse global region graph. Rejected for v1 (author chose decorative). It could be added later as an alternative river op (0006).
- Lakes by depression-filling: needs neighbourhood/global data.

## Consequences
- Rivers may occasionally run uphill, dead-end, or form loops. This is accepted.
- `waterDistance` needs an apron (max erosion falloff distance), but because every water source is pointwise, the apron just means evaluating the same pure functions over a padded area.
- **Supersedes the rivers part of 0006**, which assumed a region grid for rivers. The region-grid mechanism is still kept for points of interest (0016).
