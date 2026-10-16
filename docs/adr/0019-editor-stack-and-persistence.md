# 0019. Editor stack: React + Vite; configs as JSON files with autosave; string/number seeds

- **Status:** Accepted
- **Date:** 2026-10-16

## Decision
- **Editor:** React + TypeScript + Vite. The map view is rendered to a `<canvas>` from chunk bitmaps produced by Web Workers. Use whatever state library the implementer prefers; keep it light (e.g. Zustand).
- **Persistence:** autosave the working config(s) to browser storage, plus explicit **Save / Open .json** so configs can be committed to the game repo. No backend.
- **Seed:** the user enters any string or number. Core hashes it to a 32-bit unsigned integer, and both the original input and the hash are stored in the config. A re-roll button generates a random new seed (UI-only randomness is fine). Each layer derives its own sub-seed as hash(seed, layer id), so reordering layers doesn't change other layers' output.

## Consequences
- Configs must be validated on load (schema + version), with migrations for older `configVersion`s.
- Layer ids must be stable (generated once, never derived from position).
