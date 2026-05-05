# BASE Pack — Vanilla Clone Plan

## What we have
- `worldgen/` = full vanilla 1.21 datapack extracted (65 biomes, 35 density functions, 60 noises, 204 configured features, 238 placed features, 34 structures, 20 structure sets, 4 carvers)
- Terra NMS delegate already delegates `applyBiomeDecoration` to vanilla → **features + structures work for free**
- `applyCarvers` is a no-op in the NMS delegate → **caves do NOT work yet**

---

## Work items (rough priority order)

### 1. Biome provider — HARD, new addon likely needed
Vanilla uses **multi-noise** (temperature, humidity, continentalness, erosion, weirdness, depth splines).
No Terra multi-noise biome provider addon exists. Options:
- **A)** Write new `biome-provider-multi-noise` addon that reads the vanilla parameter lists from `worldgen/multi_noise_biome_source_parameter_list/`. Best fidelity.
- **B)** Approximate using `biome-provider-pipeline-v2` with noise layers mapped to vanilla axes. Faster but imperfect placement.

Recommendation: **Option A** for true clone.

### 2. Terrain shape — HARD
Vanilla terrain = layered density function graph (`offset` -> `factor` -> `jaggedness` -> `sloped_cheese` -> `final_density`). These are spline-based with continentalness/erosion/weirdness inputs.

Need to translate each density function to Terra `EXPRESSION` noise samplers, OR implement a `density-function` addon that evaluates the vanilla JSON graph natively.

Key functions to port from `worldgen/density_function/overworld/`:
- `continents`, `erosion`, `ridges`, `ridges_folded`, `depth`, `offset`, `factor`, `jaggedness`, `sloped_cheese`
- All 60 noise parameter sets from `worldgen/noise/`

### 3. Surface palettes — MEDIUM
`buildSurface` is a no-op in Terra — vanilla surface builders (grass/dirt, sand beaches, etc.) are bypassed. Need per-biome `palette:` blocks in BIOME yamls for all 65 biomes.

### 4. Cave carvers — MEDIUM, requires NMS fix first
`applyCarvers` is hardcoded to no-op in `NMSChunkGeneratorDelegate`. Need a `vanilla.caves` guard analogous to the `disable.structures` fix, then set `vanilla.caves: true` in pack.yml.

Carvers to enable: `cave`, `cave_extra_underground`, `canyon`.

### 5. Biome YAML files — MEDIUM/TEDIOUS
One `.yml` per biome (65 total). Each needs:
- `vanilla: minecraft:<biome>` mapping
- `terrain:` sampler referencing shared noise expressions
- `palette:` surface layers

Most overworld biomes share the same noise — parameterise, don't duplicate.

### 6. Pack wiring — SMALL
- Switch `biome-provider-single` to new multi-noise provider
- Add `vanilla.caves: true` once NMS delegate is fixed
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
5. **Terrain linkage**: Verify each biome generates terrain at correct height (plains flat, jagged peaks tall) via terrain sampler from Work Item 3.