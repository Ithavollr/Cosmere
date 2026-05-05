# BASE Pack — Biome Provider Architecture

## Overview

This document captures the design decisions and implementation structure for the vanilla-like biome approximation in the BASE pack. The system uses Terra's existing `biome-provider-pipeline-v2` and `biome-provider-extrusion` addons, configured entirely via YAML. No custom Java code was written.

## MISSING FEATURES AND GAPS CAUSED BY THE DUMBASS AI:

### Missing Biome YAMLs (No Pipeline Stage Can Generate Them)

The following vanilla overworld biomes exist in `worldgen/biome/` but have **no corresponding `.yml` file** and **no pipeline stage** that can produce them:

- ~~**Coast / Shore biomes**~~: `beach`, `snowy_beach`, `stony_shore` — Fixed. `COAST` continental tier added; temperature stage resolves to final biome IDs.
- **River biomes** (require a dedicated river noise sampler and carve stage):
  - `river`
  - `frozen_river`
- ~~**Mushroom biome**~~: `mushroom_fields` — Fixed. Added as most-negative continentalness slice in the source.
- **1.20+ / Variant biomes** missing from all humidity splits:
  - `cherry_grove`
  - `pale_garden`
  - `mangrove_swamp`
  - `flower_forest`
  - `sunflower_plains`
  - `old_growth_birch_forest`
  - `old_growth_pine_taiga`
  - `sparse_jungle`
  - `bamboo_jungle`
  - `savanna_plateau`
  - `windswept_savanna`
  - `wooded_badlands`
  - `eroded_badlands`
  - `grove`
  - `snowy_slopes`
  - `windswept_forest`

### Pipeline Architecture Gaps

- ~~**No coast tier**~~: Fixed. `COAST: 10` tier added to continental source between `OCEAN` and `LAND_FLAT`. Temperature stage resolves it directly to `SNOWY_BEACH` (frozen), `BEACH` (cold/temperate/warm), `STONY_SHORE` (hot/rugged weight).
- ~~**No mushroom island stage**~~: Fixed. `MUSHROOM_FIELDS: 3` added as the most-negative continentalness slice, below `DEEP_OCEAN`. No separate stage needed; the source directly produces the final biome.
- **No river stage**: Vanilla places `river` / `frozen_river` via a dedicated `river_noise` parameter that carves through land biomes. Overworld2 implements this with `add_rivers.yml` using `REPLACE_LIST` and biome tags (`USE_RIVER`, `USE_FROZEN_RIVER`). We have no river sampler, no tags on biomes, and no river stage.
- **`REPLACE_LIST` syntax unverified**: All humidity stages and the temperature stage use `type: REPLACE_LIST` with a `from:` block containing named biome keys. The Java `ReplaceMutator` class was inspected, but the `REPLACE_LIST` mutator (which handles multiple `from` entries in a single stage) was **not verified** against the actual Terra `biome-provider-pipeline-v2` addon. If `REPLACE_LIST` does not exist or uses different syntax, all stage files are broken.
- **`erosion` noise unused**: The `erosion` sampler is defined in `noise/biome-samplers.yml` and `meta.yml` but no pipeline stage references it. In vanilla, erosion drives terrain shape (via density functions), not biome selection, so this is acceptable, but the parameter is dead weight in the current pipeline.

### Biome YAML Structural Bugs (RESOLVED — Work Item 3 complete)

- ~~**`terrain.palette` nesting**~~: Fixed. All biome YAMLs now have root-level `palette:`.
- ~~**No `ocean` config in aquatic biomes**~~: Fixed. All 9 ocean YAMLs have `ocean.level: 63` and `ocean.palette: OCEAN_WATER`.
- ~~**No slant palettes**~~: Fixed. `WINDSWEPT_HILLS`, `WINDSWEPT_GRAVELLY_HILLS`, `JAGGED_PEAKS`, `STONY_PEAKS`, `FROZEN_PEAKS` all have `slant:` config.
- ~~**Missing root-level `palette:`**~~: Fixed.

### Missing Asset Files

- ~~**No `terrain/` directory**~~: Fixed. Eight abstract biomes (`TERRAIN_FLAT`, `TERRAIN_LOWLANDS`, `TERRAIN_ROLLING`, `TERRAIN_HILLS`, `TERRAIN_MOUNTAINS`, `TERRAIN_OCEAN`, `TERRAIN_DEEP_OCEAN`, `TERRAIN_CAVE`) defined under `biomes/abstract/` with inline `LINEAR_HEIGHTMAP` samplers. All 36 concrete biomes now use `extends:` to inherit terrain shape.
- ~~**No `palettes/` directory**~~: Fixed. All palette files created under `palettes/land/`, `palettes/aquatic/`, `palettes/cave/`, `palettes/strata/`.
- ~~**`meta.yml` missing strata anchors**~~: Fixed. `meta.yml` now has `strata:` and `palette-bottom:` blocks.

### Stale Configuration

- `meta.yml` still contains `land_threshold: 0.0`, which was used by the old 2-way continental source. The new 5-way source ignores this parameter, making it dead config.


## Design Decisions

### 1. Layered Provider Architecture

We use a **two-tier provider stack**:

- **Top layer: `EXTRUSION` provider** (`VANILLA_3D`)  
  Handles 3D cave biomes (`deep_dark`, `dripstone_caves`, `lush_caves`) by layering Y-level dependent replacements on top of the surface pipeline.

- **Base layer: `PIPELINE` provider** (`VANILLA_PIPELINE`)  
  A 2D biome resolver that runs a sequence of noise-driven `REPLACE` stages to map (x, z) coordinates to surface biomes.

This separation lets surface biomes and cave biomes be configured independently. The extrusion provider queries the pipeline at (x, z) for the base biome, then checks extrusion layers in order for that Y level.

### 2. Five Tunable Noise Axes

Vanilla Minecraft uses `continentalness`, `temperature`, `humidity` (vegetation), `erosion`, and `weirdness` (ridges) to place biomes. We approximate each with an `EXPRESSION` sampler wrapping `open_simplex_2`:

| Axis | Scale | Purpose |
|------|-------|---------|
| `continental` | x/2048, z/2048 | Ocean vs. land; continent size |
| `temperature` | x/1024, z/1024 | Frozen → hot bands |
| `humidity` | x/1024, z/1024 | Arid → humid variants within each temp band |
| `erosion` | x/512, z/512 | Flat terrain → hills |
| `weirdness` | x/512, z/512 | Valleys → peaks |

All five samplers clamp to globally tunable `floor` and `ceiling` values defined in `meta.yml`. This allows single-parameter world-type control (e.g., set `temperature_floor = temperature_ceiling = 0.8` for an all-hot world).

### 3. Pipeline Stage Sequence

The pipeline resolves biomes through **8 ordered stages**, all using `REPLACE_LIST` to subdivide multiple parent biomes simultaneously. This matches the Overworld2 pattern and approximates vanilla's multi-dimensional parameter space better than sequential `REPLACE`.

| Stage | Driver | Description |
|-------|--------|-------------|
| Source | `continental` | 7-way split: `MUSHROOM_FIELDS` (3%), `DEEP_OCEAN` (27%), `OCEAN` (15%), `COAST` (10%), `LAND_FLAT` (20%), `LAND_HILLS` (20%), `LAND_MOUNTAINS` (5%). Continentalness directly drives both ocean depth and land elevation. `MUSHROOM_FIELDS` occupies the most-negative slice. |
| 01 | `temperature` | Splits **all** continental tiers by temperature simultaneously. Ocean/coast tiers resolve to final biome IDs; land tiers produce intermediate placeholders (`flat-frozen` … `mountains-hot`). `COAST` resolves to `SNOWY_BEACH` (frozen) / `BEACH` (all others) by temperature weight. |
| 02 | `humidity` | Splits frozen land tiers (`flat-frozen`, `hills-frozen`, `mountains-frozen`) into final biomes. Frozen mountains resolve directly to peak biomes. |
| 03 | `humidity` | Splits cold land tiers into final biomes. Cold mountains resolve directly to `JAGGED_PEAKS`. |
| 04 | `humidity` | Splits temperate land tiers into final biomes. Temperate mountains resolve to `MEADOW` (placeholder for weirdness). |
| 05 | `humidity` | Splits warm land tiers into final biomes. Warm mountains resolve to `MEADOW`. |
| 06 | `humidity` | Splits hot land tiers into final biomes. Hot mountains resolve to `MEADOW`. |
| 07 | `weirdness` | Splits `MEADOW` (from temperate/warm/hot mountains) into `MEADOW` (low weirdness) and `STONY_PEAKS` (high weirdness). |
| 08 | fine-tuning | `BORDER`/`BORDER_LIST`/`REPLACE_LIST` passes using `terra:` tags. All biomes are fully resolved at this point. Currently: replaces `terra:beach` adjacent to `terra:rugged` with `STONY_SHORE`. Future entries: variant biomes, specialty replacements. |

Each stage file lives in `biome-providers/stages/` with an ordered prefix (`01-`, `02-`, …) to ensure deterministic execution order.

### Fine-Tuning Stage Tag Conventions

Biome YAMLs may carry `tags:` entries using the `terra:` namespace. These are available to `BORDER`/`BORDER_LIST` stages but have no effect on earlier stages. The `BIOME:<ID>` and `ALL` tags are always auto-added by Terra regardless.

| Tag | Biomes | Purpose |
|-----|--------|---------|
| `terra:rugged` | `WINDSWEPT_HILLS`, `WINDSWEPT_GRAVELLY_HILLS`, `JAGGED_PEAKS`, `STONY_PEAKS`, `FROZEN_PEAKS` | Marks elevated/rocky terrain for stony shore adjacency and future cliff replacements |
| `terra:beach` | `BEACH`, `SNOWY_BEACH` | Marks sandy/snowy coast for replacement when adjacent to rugged terrain |

### 4. Vanilla Biome ID Mapping

Every biome definition includes a `vanilla: minecraft:XXX` key. This is **critical for NMS decoration passthrough**: when `NMSChunkGeneratorDelegate.applyBiomeDecoration()` runs, it reads the vanilla biome ID and places the correct vanilla structures, ores, and vegetation for that biome. Terra's custom trees run via the Bukkit populator in parallel.

### 5. Tree Feature Assignments

Existing Terra tree features (`features/vegetation/trees/*.yml`) were mapped to each biome by adding a `features:` list to the biome definition:

| Biome | Assigned Tree Features |
|-------|------------------------|
| `PLAINS` | `TEMPERATE_TREES` |
| `FOREST` | `TEMPERATE_TREES`, `DENSE_TEMPERATE_TREE_PATCHES` |
| `BIRCH_FOREST` | `BIRCH_TREES` |
| `JUNGLE` | `JUNGLE_TREES`, `MANGROVE_TREES` |
| `SAVANNA` | `ACACIA_TREES`, `SPARSE_ACACIA_TREES` |
| `DESERT` | `DEAD_TREES_SPARSE` |
| `SNOWY_TAIGA`, `TAIGA` | `SPRUCE_TREES`, `SPARSE_SPRUCE_TREES` |
| `OLD_GROWTH_SPRUCE_TAIGA` | `GIANT_REDWOODS`, `EVERGREEN_TREES` |
| `DARK_FOREST` | `DENSE_DARK_FOREST_TREES` |
| `SWAMP` | `SWAMP_TREES`, `DEAD_SWAMP_TREES` |
| `BADLANDS` | `SPARSE_OAK_TREES` |
| `MEADOW` | `SEASONAL_TREES`, `GREAT_AZALEA_TREES` |
| `SNOWY_TUNDRA` | `SPARSE_SPRUCE_TREES`, `ICE_SPIKES` |
| `ICE_SPIKES` | `ICE_SPIKES` |
| Peaks (`JAGGED`, `STONY`, `FROZEN`) | `[]` (no trees) |

## Implementation Limitations

### Rivers
Vanilla places rivers via a dedicated `river_noise` parameter that carves through land biomes. Terra's pipeline does **not** have a native "river carve" stage. The initial implementation approximates wetlands via humidity-based `SWAMP` placement. True river generation requires adding a `river` noise sampler and an early pipeline stage that replaces a narrow noise band with `RIVER` biome.

### Noise Library Differences
Terra's `open_simplex_2` does **not** match vanilla's `shifted_noise` / `flat_cache` / `weirdness` spline curves exactly. The scales (`/2048`, `/1024`, `/512`) and salt offsets were chosen to produce similar continent sizes and climate patch sizes, but the resulting biome boundaries will differ from vanilla's exact placement.

### Pipeline Stage Syntax
Each `REPLACE` stage uses a `ProbabilityCollection<PipelineBiome>` driven by a scalar sampler value. The syntax `FROZEN_OCEAN: 0.15` means "if sampler value ≤ 0.15, return `FROZEN_OCEAN`". Cumulative ordering determines band boundaries. This is functionally equivalent to threshold ranges but expressed as scalar cutoffs. The exact boundary behavior depends on Terra's `ProbabilityCollection` implementation.

### 3D Cave Biomes
The `EXTRUSION` provider uses `min-y` / `max-y` bounds and a binary noise sampler (`raw > 0.6 ? 1 : 0`) to place cave biomes. This is a **discrete on/off** placement rather than vanilla's gradual cave biome blending. `deep_dark` uses a stricter threshold (`> 0.6`) to ensure it is rarer than `dripstone_caves` and `lush_caves`.

## File Structure

```
BASE/
├── meta.yml                          # Global tunable parameters
├── noise/
│   └── biome-samplers.yml            # 5 surface + 3 cave noise samplers
├── biome-providers/
│   ├── extrusion.yml                 # Top-level VANILLA_3D provider
│   ├── pipeline.yml                  # Base VANILLA_PIPELINE provider
│   ├── sources/
│   │   └── continental-source.yml    # Ocean/land split source
│   └── stages/
│       ├── 01-temperature.yml          # Splits all continental tiers by temperature (REPLACE_LIST)
│       ├── 02-humidity-frozen.yml      # Splits frozen land tiers by humidity (REPLACE_LIST)
│       ├── 03-humidity-cold.yml        # Splits cold land tiers by humidity (REPLACE_LIST)
│       ├── 04-humidity-temperate.yml   # Splits temperate land tiers by humidity (REPLACE_LIST)
│       ├── 05-humidity-warm.yml        # Splits warm land tiers by humidity (REPLACE_LIST)
│       ├── 06-humidity-hot.yml         # Splits hot land tiers by humidity (REPLACE_LIST)
│       └── 07-weirdness-peaks.yml      # Splits mountain meadows into peaks (REPLACE_LIST)
├── biomes/
│   ├── aquatic/
│   │   ├── frozen_ocean.yml
│   │   ├── cold_ocean.yml
│   │   ├── ocean.yml
│   │   ├── lukewarm_ocean.yml
│   │   ├── warm_ocean.yml
│   │   ├── deep_frozen_ocean.yml
│   │   ├── deep_cold_ocean.yml
│   │   ├── deep_ocean.yml
│   │   └── deep_lukewarm_ocean.yml
│   ├── land/
│   │   ├── plains.yml
│   │   ├── forest.yml
│   │   ├── birch_forest.yml
│   │   ├── jungle.yml
│   │   ├── savanna.yml
│   │   ├── desert.yml
│   │   ├── snowy_taiga.yml
│   │   ├── taiga.yml
│   │   ├── old_growth_spruce_taiga.yml
│   │   ├── dark_forest.yml
│   │   ├── swamp.yml
│   │   ├── badlands.yml
│   │   ├── windswept_hills.yml
│   │   ├── windswept_gravelly_hills.yml
│   │   ├── meadow.yml
│   │   ├── jagged_peaks.yml
│   │   ├── stony_peaks.yml
│   │   ├── frozen_peaks.yml
│   │   ├── snowy_tundra.yml
│   │   └── ice_spikes.yml
│   └── cave/
│       ├── deep_dark.yml
│       ├── dripstone_caves.yml
│       └── lush_caves.yml
└── pack.yml                          # References VANILLA_3D provider
```

## Verification Checklist

- [ ] Unit test: All-ocean world (`continental_floor = continental_ceiling = -0.5`)
- [ ] Unit test: All-hot world (`temperature_floor = temperature_ceiling = 0.8`)
- [ ] Integration test: Fly across X/Z verifying latitudinal biome transitions
- [ ] 3D test: Descend below Y=0 verifying cave biome placement
- [ ] Terrain linkage: Verify terrain samplers from Work Item 3 produce correct heights per biome
