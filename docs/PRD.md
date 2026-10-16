# PRD: Map Maker — procedural terrain generator & editor

- **Status:** Ready for implementation
- **Date:** 2026-10-16
- **Owner:** Mike Malcolm (sole user)
- **Decisions:** see [`docs/adr/`](adr/README.md). ADR numbers are cited as (ADR-NNNN). If this PRD and an ADR conflict, the ADR wins; flag the conflict rather than guessing.

---

## 1. Summary

Build a **TypeScript monorepo** with two parts:

1. **`packages/core`**: a pure, deterministic library that generates an **infinite, chunked, 2D top-down tile world** from a JSON **config** (seed + parameters). The author's TypeScript game imports this package and regenerates exactly the same terrain at runtime.
2. **`apps/editor`**: a single-user **React + Vite web app** for designing those configs. It provides a layer stack of terrain features, plain-language dials, an exact live preview with pan/zoom, side-by-side A/B comparison, editable terrain types, data maps, and points of interest. It also edits separate **cave/mine maps** with the same tools.

The deliverable of the editor is a **config JSON file**, not images (ADR-0003).

## 2. Goals

- G1. Design surface world maps and cave maps by stacking and tuning features (elevation noise, water, erosion, climate, terrain types, data maps, POIs) with immediate visual feedback.
- G2. Output is a small JSON config. The game calls `generateChunk(config, cx, cy)` and gets **bit-identical** results to the editor preview.
- G3. Infinite world: any chunk can be generated independently and chunks join **seamlessly**.
- G4. Easy experimentation: toggle, reorder, and swap layers, compare two variants side by side, undo/redo.

## 3. Non-goals (v1)

- Player edits / persisted world changes (the game layers diffs on top).
- Linking mine-entrance POIs to cave configs (the game derives the cave seed; ADR-0005).
- Node-graph editor UI (but the config is graph-shaped; ADR-0007).
- PNG/heightmap/image export, 3D preview, hillshade.
- Physically correct river flow or full hydraulic simulation (ADR-0010, ADR-0012).
- Backend, accounts, multi-user, mobile layout, i18n, accessibility beyond sensible defaults (ADR-0002).

## 4. Glossary

| Term | Meaning |
|---|---|
| **Cell / tile** | One grid square. 1 cell = 1 game tile (ADR-0017). Integer world coords `(x, y)`. |
| **Chunk** | Square block of `chunkSize × chunkSize` cells (default 64) at chunk coords `(cx, cy)`. Covers cells `[cx*S, cx*S+S) × [cy*S, cy*S+S)`. |
| **Field** | A named `Float32Array` scalar grid over a region (elevation, temperature, moisture, waterDistance, …). |
| **Operation (op)** | A pluggable unit of generation with typed inputs/outputs, dials, advanced params, and an **apron** (ADR-0006). |
| **Apron** | Number of extra border cells an op needs around the requested region to compute its output exactly. |
| **Layer** | An op instance as shown in the editor's layer stack. |
| **Dial** | User-facing, plain-language control (e.g. Strength 0–100) mapped to raw params (ADR-0009). |
| **Map config** | JSON document describing one map (surface or cave). |
| **Terrain type** | Editor-defined class (id, name, colour, rules) assigned to each cell (ADR-0014). |
| **Data layer** | Per-cell resource / vegetation / spawn-zone output (ADR-0016). |
| **POI** | Sparse point of interest placed via a deterministic region grid (ADR-0016). |

## 5. Architecture

### 5.1 Repo layout

```
/
├─ package.json            # pnpm workspaces
├─ pnpm-workspace.yaml
├─ packages/
│  └─ core/                # @map-maker/core — zero runtime deps preferred, no DOM
│     ├─ src/
│     │  ├─ config/        # types, JSON schema/validation, migrations, seed hashing
│     │  ├─ math/          # hash, PRNG, deterministic approximations (no Math.sin etc.)
│     │  ├─ noise/         # simplex/OpenSimplex2, fBm, ridged, billow, domain warp
│     │  ├─ ops/           # op definitions (one file per op) + registry
│     │  ├─ eval/          # graph compile, apron propagation, region evaluation, cropping
│     │  ├─ classify/      # terrain type rules
│     │  ├─ data/          # data layers, POIs
│     │  ├─ templates/     # default "Surface" and "Cave" configs
│     │  └─ index.ts       # public API
│     └─ test/
├─ apps/
│  └─ editor/              # React + Vite + TS
└─ docs/ (PRD.md, adr/)
```

Tooling: TypeScript strict, Vitest, ESLint + Prettier, Playwright for cross-browser determinism and a smoke e2e test. Node 22+ (current LTS fine).

### 5.2 Determinism rules (ADR-0015). Hard requirements.

- All randomness comes from **seeded integer hashing** (e.g. a 32-bit avalanche hash such as PCG/xxHash-style mixing over `(seed, x, y, salt)` using `Math.imul` and `>>> 0`). No `Math.random`, `Date`, `crypto`, or reliance on object-key / `Map` iteration order for anything that affects output.
- In generation code, use **only IEEE-exact operations**: `+ - * /`, `Math.sqrt`, `Math.floor/ceil/round/trunc`, `Math.abs`, `Math.min/max`, `Math.fround`, `Math.imul`, bitwise ops. **Forbidden:** `Math.sin/cos/tan/atan2/exp/log/pow/hypot/cbrt` etc. If needed, implement deterministic polynomial approximations in `core/math`. Enforce with an ESLint `no-restricted-properties` rule scoped to `packages/core/src`.
- CPU only. No GPU in core.
- **Seamlessness invariant:** a cell's value must not depend on which chunk or region requested it. `generateChunk(c, cx, cy)` must equal the matching slice of `generateRegion(c, biggerRect)`. Each op must be exact within its declared apron. Distance-based ops **clamp** at their max distance so results don't depend on data beyond the apron.
- Seeds: user input (string or number) → `hashSeed(input): uint32`. Each layer's sub-seed is `hash(seed, stableLayerId)`, so reordering layers never changes other layers' output (ADR-0019).
- Any change to core that alters output for an existing config requires bumping op `version` and/or `configVersion`, plus a migration or explicit golden update.

### 5.3 Config schema (v1)

The shape is normative; field names may be adjusted if documented. Ship a runtime validator (hand-written or zod; if zod, keep it out of the hot path) and a JSON Schema file.

```ts
type ConfigVersion = 1;

interface MapConfig {
  configVersion: ConfigVersion;
  id: string;                     // stable uuid
  name: string;                   // "Overworld", "Iron Mine"
  kind: 'surface' | 'cave';       // template label; does not change evaluation semantics
  seed: { input: string; hash: number };  // hash = hashSeed(input)
  chunkSize: number;              // default 64, power of two 16..256
  graph: {
    nodes: OpNode[];              // evaluation DAG (ADR-0007)
    layerOrder: string[];         // node ids in editor stack order (UI only)
    outputs: {                    // which node/port provides canonical fields
      elevation: PortRef;
      water?: PortRef;            // water kind: 0 none, 1 ocean, 2 river, 3 lake
      waterDistance?: PortRef;
      temperature?: PortRef;
      moisture?: PortRef;
      [extra: string]: PortRef | undefined;
    };
  };
  terrainTypes: TerrainType[];    // ADR-0014
  dataLayers: DataLayer[];        // ADR-0016
  poiTypes: PoiType[];            // ADR-0016
  poiRegionSize: number;          // default 256
  editor?: { viewCenter?: [number, number]; zoom?: number; viewMode?: string }; // ignored by core
}

interface PortRef { node: string; port: string }

interface OpNode {
  id: string;                     // stable, generated once (never index-based)
  op: string;                     // registry id, e.g. "noise", "blend", "coastalErosion"
  opVersion: number;
  label: string;                  // user-editable layer name
  enabled: boolean;               // disabled → passthrough of primary input (op defines which)
  inputs: Record<string, PortRef>;
  dials: Record<string, number>;
  advanced?: Record<string, number | string | boolean>; // overrides; absent = derived from dials
}

interface TerrainType {
  id: number;                     // stable uint16, never reused
  name: string;
  color: string;                  // #rrggbb
  priority: number;               // higher wins
  rules: Rule[];                  // all must match (AND); empty = always matches
  fallback?: boolean;             // exactly one type must be fallback
}

type Rule =
  | { field: string; min?: number; max?: number }        // any field by name
  | { water: Array<'none' | 'ocean' | 'river' | 'lake'> }
  | { terrainType: number[] };                            // for data layers/POIs only

interface DataLayer {
  id: string;                     // stable
  name: string;                   // "Iron", "Trees", "Wolf spawns"
  kind: 'resource' | 'vegetation' | 'spawn';
  output: 'density' | 'id';       // density: Float32 0..1; id: Uint8 (0 = none)
  entries: Array<{                // for 'id' outputs: candidate ids; for density: one entry
    id: number; name: string; color: string;
    rules: Rule[];
    noise: NoiseDials;            // scale, threshold/strength, roughness
    seedSalt: string;
  }>;
}

interface PoiType {
  id: number; name: string; color: string; icon?: string;
  chancePerRegion: number;        // 0..1
  maxPerRegion: number;           // default 1
  rules: Rule[];                  // evaluated at candidate cell
  minDistanceFromWater?: number;
  isMineEntrance?: boolean;       // informational for the game
}
```

### 5.4 Core public API

```ts
hashSeed(input: string | number): number;
validateConfig(json: unknown): { ok: true; config: MapConfig } | { ok: false; errors: string[] };
migrateConfig(json: unknown): MapConfig;          // older configVersion → current
compile(config: MapConfig): CompiledMap;          // resolves registry, topo-sorts, computes aprons; throws on cycles/missing ops

generateChunk(map: CompiledMap, cx: number, cy: number, opts?: {
  fields?: string[];          // extra intermediate fields to return (editor raw views); default canonical only
  signal?: AbortSignal;
}): ChunkData;

generateRegion(map: CompiledMap, rect: { x: number; y: number; w: number; h: number }, opts?): RegionData; // used by tests & sampleCell

sampleCell(map: CompiledMap, x: number, y: number): CellInfo; // all field values + matched terrain type + data layer values (editor inspector)

poisInChunk(map: CompiledMap, cx: number, cy: number): Poi[]; // also included in ChunkData

interface ChunkData {
  cx: number; cy: number; size: number;
  terrain: Uint16Array;                         // terrain type id per cell, row-major
  fields: Record<string, Float32Array>;         // canonical + requested
  data: Record<string, Float32Array | Uint8Array>; // by dataLayer id
  pois: Poi[];                                  // whose position lies in this chunk
}
interface Poi { typeId: number; x: number; y: number; regionX: number; regionY: number; seed: number } // seed = hash(mapSeed, x, y, typeId) — the game may use it for cave seeds
```

Typed arrays must be transferable (no shared views) so the editor and game can post them across workers.

### 5.5 Operation interface (ADR-0006, ADR-0009)

```ts
interface OpDefinition<P> {
  id: string; version: number; label: string;
  category: 'elevation' | 'water' | 'erosion' | 'climate' | 'combine' | 'cave' | 'utility';
  inputs: PortSpec[];           // { name, required, description }
  outputs: PortSpec[];
  passthrough?: { input: string; output: string }; // behaviour when disabled
  dials: DialSpec[];            // { key, label, min, max, step, default, help }
  advanced: ParamSpec[];        // raw params, shown behind "Advanced"
  dialsToParams(dials: Record<string, number>, advanced: Partial<P>): P; // advanced overrides win
  apron(params: P): number;     // 0 for pointwise ops
  evaluate(ctx: {
    rect: Rect;                 // region to produce (already includes downstream aprons)
    inputs: Record<string, Float32Array>; // each sized to rect grown by this op's apron
    inputRect: Rect;
    seed: number;               // layer sub-seed
    params: P;
  }): Record<string, Float32Array>; // each sized to rect
}
```

**Evaluation algorithm:** compile the DAG; for the requested chunk rect, walk from outputs to sources accumulating aprons, so each node gets the rect it must produce (union if a node feeds several consumers). Evaluate in topological order, then crop to the chunk. Provide a `pointwise()` helper for ops that compute `f(x, y, inputsAt(x, y))`.

### 5.6 Field conventions

- `elevation`: nominal `0..1` (0 = deepest, 1 = highest peak). Ops may exceed this temporarily; the final elevation is clamped to `0..1`.
- `temperature`, `moisture`: `0..1`.
- `water`: stored as Float32 codes `0|1|2|3` (none/ocean/river/lake). Exposed to rules by name.
- `waterDistance`: tiles to the nearest water cell, **clamped** to the max distance any consumer needs (≤ apron).
- Sea level is a dial on the Water op, nominal `0..1`, default ~0.4.

## 6. Operation catalogue (v1)

Dials are the defaults shown. Advanced params are listed after "Adv:". Ranges are guidance; tune for feel.

### Elevation / combine
| Op | Inputs → outputs | Dials | Adv | Apron |
|---|---|---|---|---|
| `noise` | (optional `mask`) → `out` | Scale (feature size in tiles, 8–4000), Strength (0–1), Roughness (0–1 → gain), Detail (1–8 octaves) | style: fbm/ridged/billow; lacunarity; offset; warp strength & scale; noise algo (simplex default) | 0 |
| `blend` | `a`, `b`, (optional `mask`) → `out` | Amount (0–1) | mode: add / subtract / multiply / max / min / lerp | 0 |
| `remap` | `in` → `out` | Contrast, Bias | curve points (piecewise linear), terraces count | 0 |
| ~~`falloff`~~ | **Do not implement**: an island falloff mask needs a global centre (ADR-0004). | | | |

The default Surface template stacks: Continents (large scale, strong), Mountains (ridged, masked by continents), Hills, Detail.

### Water (ADR-0012)
| Op | Inputs → outputs | Dials | Adv | Apron |
|---|---|---|---|---|
| `water` | `elevation` → `elevation`, `water`, `waterDistance` | Sea level | max distance (auto = max needed by downstream ops) | maxDistance |
| `rivers` | `elevation` → `elevation`, `riverMask` | Amount, Width, Winding, Depth | max elevation for rivers; coast blend distance; noise scale | 0 |
| `lakes` | `elevation` → `elevation`, `lakeMask` | Amount, Size | basin noise scale; level smoothing | 0 |

Order in the default template: elevation stack → `rivers` → `lakes` → `water` (which merges ocean, riverMask, and lakeMask into `water` and computes `waterDistance`) → `coastalErosion` → `water` recomputation if erosion changed classification (or erosion preserves water classification; see below).

River algorithm (decorative): `n = warpedNoise(x, y)`, river where `|n| < width'`; carve `elevation -= depth * smoothstep`. Suppress above max elevation; blend to sea level near the coast. Rivers may form loops or run uphill; that's acceptable.

Lake algorithm: low-frequency basin noise picks lake regions. Inside a region, the lake surface level is a smooth, near-constant function (e.g. the basin noise's value quantised per region with smooth blending). Cells below that level are lake. Water surfaces must look flat (one colour per lake in the classified view).

### Erosion (ADR-0010)
| Op | Inputs → outputs | Dials | Adv | Apron |
|---|---|---|---|---|
| `coastalErosion` | `elevation`, `water`, `waterDistance` → `elevation` | Strength (0–100) | falloff distance (default scales 0→24 tiles with Strength); max lowering; beach slope; edge noise amount/scale; river widen factor | falloff distance |

Behaviour as Strength rises: (1) land near the sea is lowered and flattened toward sea level, widening the beach band; (2) land near any water is lowered (floodplains, shallows, wetland conditions); (3) river channels widen and deepen. **It must not flip cells between land and water**: eroded land is clamped to ≥ waterLevel + ε for that water body. The water classification from the `water` op stays authoritative.

### Climate
| Op | Inputs → outputs | Dials | Adv | Apron |
|---|---|---|---|---|
| `temperature` | `elevation` → `temperature` | Scale, Average, Variation, Altitude cooling | lapse curve; latitude-style banding off by default (infinite world) | 0 |
| `moisture` | `waterDistance` (opt) → `moisture` | Scale, Average, Variation, Wetness near water | falloff distance | 0 (uses already-computed waterDistance) |

### Cave ops (ADR-0005)
| Op | Inputs → outputs | Dials | Adv | Apron |
|---|---|---|---|---|
| `threshold` | `in` → `out` (0/1) | Level | soft edge | 0 |
| `cellularSmooth` | `in` (0/1) → `out` | Passes (0–8), Openness | birth/survive neighbour counts | passes |
| `distance` | `in` (0/1) → `out` | Max distance | — | max distance |

The default Cave template: noise (+ warp) → threshold → cellularSmooth → distance-to-wall field. Terrain types: Wall, Floor, Deep floor, Water pool. Data layers: ore veins near walls. POIs: treasure, exit.

### Classification & data (not graph ops; run after the graph)
- **Classify:** for each cell, pick the highest-priority `TerrainType` whose rules all match, otherwise the fallback. Optional per-type `dither` (Adv) adds hashed noise to rule bounds to break up contour lines.
- **Data layers:** for each entry, `rulesMatch(cell) ? noiseValue(entry) : 0`. `density` outputs keep the value 0..1; `id` outputs pick the first matching entry whose noise exceeds its threshold.
- **POIs:** for each region intersecting the chunk and each POI type: `h = hash(seed, rx, ry, typeId, k)`; if `h < chancePerRegion`, place at a jittered position inside the region (inset by a margin to keep spacing), evaluate rules via the field values at that cell, and keep the POI if valid. Return only POIs inside the chunk.

## 7. Editor requirements

### 7.1 Layout

```
┌──────────────────────────────────────────────────────────────────────────┐
│ [Map: Overworld ▾][+ new map]  Seed:[oakvale][🎲]  View:[Terrain ▾]  Overlays▾ │
│ [Compare A|B]  [Undo][Redo]  [Open][Save]                                     │
├───────────────┬──────────────────────────────────────────┬───────────────┤
│ LAYERS        │                                          │ SELECTED LAYER │
│ ☑ Continents  │                                          │ Mountains      │
│ ☑ Mountains ◀ │            MAP CANVAS                    │ Scale   ──●──  │
│ ☑ Hills       │       (pan: drag, zoom: wheel)           │ Strength ─●─── │
│ ☐ Detail      │                                          │ Roughness ──●─ │
│ ☑ Rivers      │                                          │ [Advanced ▸]   │
│ ☑ Lakes       │                                          │ Algorithm ▾    │
│ ☑ Water       │                                          │                │
│ ☑ Erosion     │                                ┌────────┐│                │
│ ☑ Temperature │                                │ x,y    ││                │
│ ☑ Moisture    │                                │ elev…  ││                │
│ [+ Add layer] │                                │ type   ││                │
├───────────────┤                                └────────┘│                │
│ Tabs: Terrain types | Data layers | POIs                  │                │
└───────────────┴──────────────────────────────────────────┴───────────────┘
```

### 7.2 Functional requirements

**Maps**
- E1. Several map configs in one workspace (e.g. "Overworld" surface, "Iron Mine" cave). Create from the **Surface** or **Cave** template, duplicate, rename, delete (with confirmation).

**Layer stack** (ADR-0007)
- E2. Add a layer from a categorised op picker; remove; drag to reorder; enable/disable checkbox; rename.
- E3. Inputs default to "the previous layer's output". Ops with extra inputs (e.g. `mask`, `b`) offer a dropdown of earlier layers' outputs. Reordering that would break a reference shows an inline error, not a crash.
- E4. Selecting a layer shows its **dials** as sliders with numeric entry and help tooltips. An **Advanced** disclosure shows raw params and, where several ops share a role, an **algorithm** dropdown that swaps the op while keeping compatible dial values. Advanced overrides show a "reset to dial-derived" control.
- E5. Each layer row shows its apron/cost indicator (e.g. "+24 tiles").

**Seed**
- E6. Seed text box accepts any string or number; the 🎲 button randomises it. Changing the seed regenerates everything.

**Preview** (ADR-0018, ADR-0020)
- E7. Canvas with drag-to-pan and wheel/pinch zoom around the cursor. Zoom range: up to about 16 px per tile zoomed in; zoomed out, capped so the visible area is ≤ ~2048×2048 tiles. Always exact: no LOD approximation.
- E8. View modes: **Terrain** (terrain type colours), **Field: <name>** for any canonical or intermediate field (grayscale; water drawn in blue for elevation view), and **Layer output** (the selected layer's output, for debugging).
- E9. Overlays (toggle each): each data layer (density as alpha, ids as colours), POI markers with type colour/icon, chunk grid with coords.
- E10. Hover inspector: world `(x, y)`, chunk `(cx, cy)`, all field values, matched terrain type, data layer values (via `sampleCell` or the cached chunk).
- E11. "Go to coordinates" input and "reset view to origin".

**Terrain types / data layers / POIs** (ADR-0014, ADR-0016)
- E12. Table editor for terrain types: add/remove/reorder (priority), name, colour picker, rules editor (field dropdown + min/max, water kinds), fallback marker. IDs are auto-assigned and never reused. Validation: exactly one fallback.
- E13. Data layer editor: create resource/vegetation/spawn layers, entries with rules and noise dials, colour.
- E14. POI type editor: name, colour, chance, max per region, rules, min distance from water, mine-entrance flag. Region size setting.

**Compare** (ADR-0011)
- E15. The Compare toggle splits the canvas into A | B (vertical split, draggable divider), with locked pan/zoom. B starts as a copy of A. The editing panel targets the selected side (clear A/B indicator). "Keep A" / "Keep B" promotes one and exits compare.

**History & persistence** (ADR-0019)
- E16. Undo/redo (Ctrl+Z / Ctrl+Shift+Z) over config changes; consecutive changes from one slider drag coalesce into one entry.
- E17. Autosave the workspace to IndexedDB/localStorage (wrapped in try/catch, so the app works if storage is unavailable). Restore on load.
- E18. **Save** downloads `<map-name>.map.json` (one config per file). **Open** loads a `.map.json`, validates and migrates it, and shows readable errors on failure.

### 7.3 Rendering & workers

- Worker pool of `max(1, hardwareConcurrency - 1)` workers, each running `@map-maker/core`.
- Main thread computes the visible chunk set, requests missing chunks **centre-out**, and draws each chunk to an offscreen `ImageBitmap`/`ImageData` as it arrives (progressive fill).
- Chunk cache: LRU keyed by `(configHash, viewMode-relevant fields, cx, cy)`. `configHash` is a stable hash of the canonical JSON of the evaluation-relevant config (excluding `editor`, labels, colours). Colour/overlay changes re-colour from cached fields without regenerating.
- On a config change: cancel in-flight jobs for stale hashes (AbortSignal / job ids), keep showing stale bitmaps until replacements arrive (no flash to blank).
- Nice to have: per-node intermediate-field caching, so changing a downstream layer doesn't recompute upstream noise.

## 8. Non-functional requirements

| ID | Requirement |
|---|---|
| N1 | **Determinism:** identical config → identical `ChunkData` bytes in Node, Chromium, and Firefox. |
| N2 | **Seamless:** chunk output equals the corresponding slice of a larger region evaluation, for all templates and both kinds of map. |
| N3 | **Performance (modern desktop, 8 cores):** median full-pipeline 64² chunk (default Surface template, including aprons) ≤ 15 ms single-thread; visible-area refresh at default zoom ≤ 200 ms after a dial change; max zoom-out full refresh ≤ 5 s. |
| N4 | Supported coordinate range: at least ±1,000,000 tiles without visible precision artefacts (offset sampling coords using integer chunk origins). |
| N5 | `@map-maker/core` has no DOM dependencies and runs in Node, browsers, and Web Workers. Tree-shakeable ESM. |
| N6 | Configs are forward-migratable (`migrateConfig`), and core refuses configs from a newer version with a clear error. |

## 9. Testing & acceptance

**Core (Vitest)**
- T1. Golden tests: for each template (Surface, Cave) × several seeds × chunk coords including negatives and far coordinates (±1e6), hash every output array and compare to committed snapshots.
- T2. Seam test: for random chunk pairs and a 3×3 block, `generateChunk` slices equal `generateRegion` over the union, **exactly**.
- T3. Order independence: reordering layers whose inputs are unchanged leaves every other layer's output unchanged (sub-seed stability).
- T4. Lint rule forbidding non-deterministic Math functions in `packages/core/src` passes.
- T5. Unit tests per op: apron correctness (op output for rect R matches when computed inside a larger rect), dial→param mapping monotonicity (e.g. more erosion Strength → wider beach band count).
- T6. Config: validate/migrate round-trip; invalid configs produce readable errors; exactly-one-fallback check.
- T7. Benchmark script (`pnpm bench`) reporting chunk ms for N3.

**Cross-runtime (Playwright)**
- T8. Run the golden suite in Chromium and Firefox via a test page, and assert the same hashes as Node.

**Editor e2e (Playwright smoke)**
- T9. Load app → default Surface map renders; drag Erosion Strength → canvas updates; toggle compare → two panes; save → file downloads; reopen → identical config hash.

**Acceptance checklist (author sign-off)**
- [ ] Create a Surface map, stack noise layers, and see continents, mountains, rivers, lakes, beaches, wetlands.
- [ ] Turning the Erosion dial visibly widens beaches, lowers land near water, and widens rivers.
- [ ] Terrain types are editable and the map recolours; hovering shows values.
- [ ] Data layers and POIs show as overlays; POIs include mine entrances.
- [ ] Create a Cave map from the template and get a believable floor/wall layout with ore.
- [ ] Compare A/B side by side and keep the better one.
- [ ] Save `.map.json`, import core into a scratch TS script, and `generateChunk` matches the editor's inspector values.
- [ ] Pan far away (e.g. x = 900,000): still works, no seams, no artefacts.

## 10. Milestones (dependency order; all are v1 scope per ADR-0020)

1. **Core foundation:** monorepo, lint rule, hash/seed, noise primitives, config types + validation, op interface, evaluator with apron propagation, `noise`/`blend`/`remap`, golden + seam test harness.
2. **Editor shell:** Vite/React app, worker pool, chunk cache, pan/zoom canvas, layer stack + dials for the elevation ops, field view mode, inspector, autosave.
3. **Water & erosion:** `rivers`, `lakes`, `water` (+ waterDistance), `coastalErosion`.
4. **Climate & classification:** `temperature`, `moisture`, terrain type table + classify + Terrain view.
5. **Data layers & POIs** + overlays.
6. **Cave ops + Cave template**, multi-map workspace.
7. **Compare A/B, undo/redo, Save/Open, migrations.**
8. **Performance pass + cross-browser determinism suite + acceptance checklist.**

## 11. Open questions / implementer latitude

- Exact noise algorithm (OpenSimplex2 vs simplex vs Perlin): implementer's choice, but it must be implemented in core (no third-party noise lib unless it obeys §5.2 and is vendored).
- Exact dial→param curves: tune for feel; document each in the op file.
- If N3 can't be met with the default erosion apron, it's acceptable to cap max falloff distance or add per-node caching. Record the choice in a new ADR.
- Any decision that contradicts an ADR: write a new ADR that supersedes it (see `docs/adr/README.md`) rather than silently diverging.
