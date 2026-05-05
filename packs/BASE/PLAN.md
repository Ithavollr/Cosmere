# BASE Pack — Vanilla Clone Plan

## What we have
- `worldgen/` = full vanilla 1.21 datapack extracted (65 biomes, 35 density functions, 60 noises, 204 configured features, 238 placed features, 34 structures, 20 structure sets, 4 carvers)
- Terra NMS delegate already delegates `applyBiomeDecoration` to vanilla → **features + structures work for free**
- `applyCarvers` is a no-op in the NMS delegate → **caves do NOT work yet**

---

## Work items (rough priority order)

### 1. Biome provider — DONE (pipeline approach)
Decision: took **Option B** — `biome-provider-pipeline-v2` with five noise axes (`continental`, `temperature`, `humidity`, `erosion`, `weirdness`) wired through `meta.yml` for world-type tunability.

7-tier continental source (`MUSHROOM_FIELDS`, `DEEP_OCEAN`, `OCEAN`, `COAST`, `LAND_FLAT`, `LAND_HILLS`, `LAND_MOUNTAINS`) feeds 8 ordered stages: temperature → 5 humidity passes → weirdness peak split → fine-tuning. The fine-tuning stage uses `BORDER` mutators against `terra:` biome tags and is the designated home for variant biomes and adjacency-driven replacements.

Multi-noise (Option A) abandoned — the pipeline gives sufficient fidelity and remains entirely YAML-configurable.

### 2. Terrain shape — DONE
Eight abstract biomes defined under `biomes/abstract/`, each with an inline `LINEAR_HEIGHTMAP` sampler. All 36 concrete biomes use `extends:` to inherit terrain shape. No separate noise files needed.

| Abstract biome | Base Y | Scale | Concrete biomes |
|---|---|---|---|
| `TERRAIN_FLAT` | 68 | 8 | plains, snowy_tundra, savanna, beach, snowy_beach, stony_shore, mushroom_fields, desert, badlands |
| `TERRAIN_LOWLANDS` | 62 | 5 | swamp |
| `TERRAIN_ROLLING` | 72 | 14 | forest, birch_forest, dark_forest, jungle, taiga, snowy_taiga, old_growth_spruce_taiga |
| `TERRAIN_HILLS` | 84 | 28 | meadow, windswept_hills, windswept_gravelly_hills, ice_spikes |
| `TERRAIN_MOUNTAINS` | 110 | 55 | stony_peaks, frozen_peaks, jagged_peaks |
| `TERRAIN_OCEAN` | 50 | 8 | ocean, lukewarm_ocean, warm_ocean |
| `TERRAIN_DEEP_OCEAN` | 34 | 8 | deep_ocean, cold_ocean, frozen_ocean, deep_cold_ocean, deep_lukewarm_ocean, deep_frozen_ocean |
| `TERRAIN_CAVE` | 320 | 0 | lush_caves, dripstone_caves, deep_dark |

Note: `base` and `scale` values are first-pass estimates; tuning against vanilla height profiles is expected during testing.

### 3. Surface palettes — DONE
19 palette YAMLs created across `palettes/{land,aquatic,cave,strata}/`. All 32 biome YAMLs use root-level `palette:` blocks, ocean biomes have `ocean.level`/`ocean.palette`, mountain/hill biomes have `slant:` configs, and `meta.yml` carries the `strata:` and `palette-bottom:` anchors.

### 4. Cave carvers — PENDING (requires NMS fix first)
`applyCarvers` is hardcoded to no-op in `NMSChunkGeneratorDelegate`. Need a `vanilla.caves` guard analogous to the `disable.structures` fix, then set `vanilla.caves: true` in pack.yml.

Carvers to enable: `cave`, `cave_extra_underground`, `canyon`.

### 5. Biome YAML files — SUBSTANTIALLY DONE (variants pending)
32 of ≈52 overworld surface biomes are written and wired through the pipeline. Remaining biomes are variants and specialty types that should be produced by `REPLACE_LIST`/`BORDER_LIST` entries in the **fine-tuning stage** (Stage 08), not by adding more continentalness/temperature/humidity slices upstream.

Variants to add (all go through fine-tuning stage):
- `cherry_grove` — `BORDER_LIST` against cold/temperate forest borders, weirdness-gated
- `pale_garden` — weirdness-driven replacement of `DARK_FOREST`
- `mangrove_swamp` — erosion-driven replacement of `SWAMP` in warm temperatures
- `flower_forest` — weirdness-driven replacement of `FOREST`
- `sunflower_plains` — weirdness-driven replacement of `PLAINS`
- `old_growth_birch_forest` — weirdness-driven replacement of `BIRCH_FOREST`
- `old_growth_pine_taiga` — weirdness-driven replacement of `TAIGA`
- `sparse_jungle`, `bamboo_jungle` — weirdness/humidity splits of `JUNGLE`
- `savanna_plateau`, `windswept_savanna` — erosion/weirdness splits of `SAVANNA`
- `wooded_badlands`, `eroded_badlands` — erosion/weirdness splits of `BADLANDS`
- `grove`, `snowy_slopes` — erosion/weirdness on cold mountain transition
- `windswept_forest` — `BORDER` between forest and `terra:rugged`

### 6. River stage — PENDING
Vanilla rivers are carved by a dedicated `river_noise` parameter that cuts through land biomes. Needs:
- New `river` sampler in `noise/biome-samplers.yml`
- New `terra:use_river` / `terra:use_frozen_river` tags on appropriate land biomes
- `REPLACE_LIST` stage gated by the river noise that turns tagged biomes into `RIVER` / `FROZEN_RIVER`
- `RIVER` and `FROZEN_RIVER` biome YAMLs and palette files

Likely placement: between weirdness (07) and fine-tuning (08), or as its own stage 09 after fine-tuning so cliffs and variants are decided before rivers carve through them.

### 7. Pack wiring — SMALL
- ~~Switch biome provider~~: Done. `pack.yml` uses `VANILLA_3D` extrusion provider wrapping `VANILLA_PIPELINE`.
- Add `vanilla.caves: true` once NMS delegate is fixed (Work Item 4)
- `disable.structures` stays `false` (default) so structures generate

---

## Already works out of the box
- All **vanilla + datapack structures** (via `applyBiomeDecoration` delegation)
- All **biome feature decorations** (ores, trees, flowers) once biome provider is wired correctly
- **Mob spawning** (`spawnOriginalMobs` delegates to vanilla)

## Will never be pixel-perfect without engine changes
- Aquifer placement (driven by vanilla density functions deep inside `fillFromNoise`)
- Ore vein density functions (`ore_veininess`, `ore_vein_a/b`) — internal to vanilla noise fill
- Terrain blending at old/new chunk borders

## Verification Plan

1. **Unit test**: Create world with `continental_floor = continental_ceiling = -0.5` → verify all chunks report ocean biomes and no land terrain generates.
2. **Unit test**: Create world with `temperature_floor = temperature_ceiling = 0.8` → verify only desert/savanna/jungle/badlands biomes appear, and vanilla cacti/acacia trees spawn.
3. **Integration test**: Default parameters world. Fly across X/Z and verify biome transitions follow latitudinal pattern: frozen poles → temperate → hot equator (with noise patchiness).
4. **3D test**: Descend below Y=0 in default world. Verify `deep_dark` biome appears in some regions (skulk blocks, wardens spawnable) and `lush_caves` / `dripstone_caves` in others.
5. **Terrain linkage**: Verify each biome generates terrain at correct height (plains flat, jagged peaks tall) via terrain sampler from Work Item 2.
6. **Coast adjacency**: Verify `STONY_SHORE` only appears where `BEACH`/`SNOWY_BEACH` directly border `terra:rugged` biomes. Plain coastlines next to plains/forest stay as sand/snow beach.
7. **Mushroom islands**: Verify `mushroom_fields` appears as small isolated patches inside deep ocean regions, never on continental land.
8. **Surface palettes**: Spot-check at least one biome from each archetype (grass, sand, snow, gravel, terracotta, calcite, slant) for correct block layering down through deepslate and bedrock strata.
9. **Ocean fill**: Verify all 9 ocean biomes fill to Y=63 with water and that beach biomes have a sub-sea sand layer down to Y=64.

## Current Status Snapshot

| Work Item | Status |
|-----------|--------|
| 1. Biome provider (pipeline) | DONE |
| 2. Terrain samplers | DONE (first-pass values, tuning pending) |
| 3. Surface palettes | DONE |
| 4. Cave carvers (needs NMS fix) | PENDING |
| 5. Variant biomes (via fine-tuning stage) | PENDING |
| 6. River stage | PENDING |
| 7. Pack wiring | DONE except `vanilla.caves` |