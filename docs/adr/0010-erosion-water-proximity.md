# 0010. Erosion: a single generation-time dial acting on land near water

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The author wants erosion as a dial that affects **land near water**, not a whole-map hydraulic simulation.

## Decision
Erosion is applied **once, at generation time**. It is part of the config and the game never re-erodes. Its main control is a **Strength** dial. Increasing it:
1. **Smooths coasts and widens beaches**: lowers and flattens slopes approaching sea level, so the sand band widens.
2. **Lowers land near water**: wider shallows, wetland/marsh zones, river floodplains.
3. **Widens and deepens river channels** and their valleys.

Implementation: the effect is driven by **distance to the nearest water** (sea, lake, river) plus elevation above water level, with a falloff whose reach and depth scale with Strength. Optional advanced params: falloff distance, max lowering, beach slope, noise breakup for natural edges.

## Alternatives considered
- Full droplet hydraulic simulation: heavier, harder to keep seamless across chunks, and not what was asked for.
- Erosion over time in-game: not required. The game can still read the dial value from the config if it ever wants to.

## Consequences
- Water must be computed **before** erosion. Pipeline order: base elevation → water (sea level, rivers, lakes) → erosion → classification.
- Distance-to-water needs neighbourhood data, so erosion declares an **apron equal to its max falloff distance** (0006). With rivers, the distance field must include river cells from neighbouring chunks, which the region-based river model supports.
- Lowering land can create new water cells. Erosion must not flood inland areas unintentionally: lowered cells keep their land/water classification, or the rule is defined explicitly.
