# 0015. Game is TypeScript: the generator ships as a shared package

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
The game regenerates maps from configs (0003), so it must produce identical output to the editor. The game is written in TypeScript/JS.

## Decision
Use a monorepo with:
- `packages/core`: the pure, UI-free, dependency-light generator (ops, evaluator, config schema, classification, data maps). The game imports exactly this package.
- `apps/editor`: the web editor, which uses `core` in Web Workers.

**Determinism rules for core:**
- All randomness comes from seeded hash functions (e.g. integer hash of seed + coords + salt). No `Math.random`, no time, no iteration-order dependence.
- Use only IEEE-754 exact operations in generation paths: `+ - * /`, `Math.sqrt`, `Math.floor`, `Math.abs`, `Math.min`/`max`, `Math.fround`, and integer ops. **Do not use** `Math.sin/cos/exp/pow/log` and similar, because their results may differ between JS engines. Use polynomial or table-based approximations implemented in core.
- Generation runs on the CPU (workers). Optional GPU paths are for preview only.

## Consequences
- Golden tests: fixed configs + chunk coords → hashed outputs, checked in CI. Any change to output is a deliberate, versioned change.
- A `configVersion` bump is needed whenever output changes for an existing config.
- The game can call the same chunk API in its own worker.
