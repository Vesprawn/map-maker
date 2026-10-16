# 0014. Terrain types are editable and the tool classifies every cell

- **Status:** Accepted
- **Date:** 2026-10-16

## Context
For a top-down tile game, something has to decide each cell's tile type. The author wants the tool to do this, and to define the types in the editor.

## Decision
Each map config has a **terrain type table**. Each entry has: stable numeric **id**, **name**, **preview colour**, and **rules**. Rules are conditions over fields (e.g. elevation range, temperature range, moisture range, water kind, water distance range) plus a **priority**. Each cell gets the highest-priority type whose rules match, with a required fallback type. The game maps type ids to its own tiles.

## Alternatives considered
- Fixed preset list: too rigid, and it can't express cave types.
- Raw values only: pushes the logic into the game and duplicates it.

## Consequences
- Type ids are part of the config contract. Renaming must not change ids.
- The editor needs a table UI for adding, editing, reordering, and recolouring types, plus a hover inspector showing each cell's field values and the matched type.
- Optional noise "dither" on rule boundaries (Advanced) avoids perfectly smooth contour lines between types.
