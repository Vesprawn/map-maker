# 0003. Output: seed + parameters, not baked images

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The tool needs to hand terrain to a game. One option is exporting baked assets (heightmap PNGs, etc.). The other is exporting a config the game uses to regenerate the terrain itself.

## Decision
The primary export is a **config (seed + generation parameters)**. The game regenerates terrain at runtime from that config. The web app is a **design-time previewer and tuner**.

## Alternatives considered
- 16-bit heightmap / splat PNG export: incompatible with an infinite world and duplicates data the game can compute.
- Raw float data export: same problem, and large.

## Consequences
- **Determinism is a hard requirement.** The same config must produce bit-identical (or tolerance-specified) output in the previewer and in the game. This rules out `Math.random()` and any unseeded randomness. It also constrains GPU-only generation, because floating-point results vary across GPUs.
- The generation core must be a **standalone, UI-free module** that the game can import.
- The config format is a public contract and needs a **schema version** for forward compatibility.
