# 0002. Platform: TypeScript browser web app, single user

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
We need an interactive way to tweak parameters and see results. The sole user is the author, who is technical.

## Decision
Build a **browser web app in TypeScript**. The audience is **just the author**, so UX polish, onboarding, accessibility hardening, accounts, and hosting are not priorities.

## Alternatives considered
- CLI: no live visual feedback while tuning.
- Rust/WASM: faster, but adds toolchain complexity before it's needed.
- Python: poor fit for an interactive browser UI.
- Desktop app: unnecessary packaging overhead.

## Consequences
- Generation must be fast enough for interactive tweaking (Web Workers and/or GPU).
- It's fine to expose raw technical parameters directly in the UI.
- No backend is needed; the app can be fully static.
