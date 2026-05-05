# BASE Pack — Biome Provider Architecture

## Overview

This document captures the design decisions and implementation structure for the vanilla-like biome approximation in the BASE pack. The system uses Terra's existing `biome-provider-pipeline-v2` and `biome-provider-extrusion` addons, configured entirely via YAML. No custom Java code was written.

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

The pipeline resolves biomes through **9 ordered stages**. Each stage uses `REPLACE` with a threshold sampler to subdivide a parent biome into child variants.

| Stage | From | Driver | Description |
|-------|------|--------|-------------|
| 1 | `OCEAN` | `temperature` | 5 ocean temperature bands (frozen → warm) |
| 2 | `LAND` | `temperature` | 5 land temperature placeholders (`LAND_FROZEN` … `LAND_HOT`) |
| 3 | `LAND_FROZEN` | `humidity` | `SNOWY_TUNDRA`, `ICE_SPIKES` |
| 4 | `LAND_COLD` | `humidity` | `SNOWY_TAIGA`, `TAIGA`, `OLD_GROWTH_SPRUCE_TAIGA` |
| 5 | `LAND_TEMPERATE` | `humidity` | `PLAINS`, `FOREST`, `BIRCH_FOREST`, `DARK_FOREST`, `SWAMP` |
| 6 | `LAND_WARM` | `humidity` | `SAVANNA`, `PLAINS`, `FOREST`, `JUNGLE` |
| 7 | `LAND_HOT` | `humidity` | `DESERT`, `SAVANNA`, `BADLANDS`, `JUNGLE` |
| 8 | `PLAINS` | `erosion` | Flat → `WINDSWEPT_HILLS` / `MEADOW` |
| 9 | `MEADOW` | `weirdness` | `JAGGED_PEAKS`, `STONY_PEAKS`, `FROZEN_PEAKS` |

Each stage file lives in `biome-providers/stages/` with an ordered prefix (`01-`, `02-`, …) to ensure deterministic execution order.

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
│       ├── 01-temperature-oceans.yml
│       ├── 02-temperature-land.yml
│       ├── 03-humidity-frozen.yml
│       ├── 04-humidity-cold.yml
│       ├── 05-humidity-temperate.yml
│       ├── 06-humidity-warm.yml
│       ├── 07-humidity-hot.yml
│       ├── 08-erosion-landforms.yml
│       └── 09-weirdness-peaks.yml
├── biomes/
│   ├── aquatic/
│   │   ├── frozen_ocean.yml
│   │   ├── cold_ocean.yml
│   │   ├── ocean.yml
│   │   ├── lukewarm_ocean.yml
│   │   └── warm_ocean.yml
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
