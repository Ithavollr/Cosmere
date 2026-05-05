# Vanilla Overworld Pack — Status

## Migrated Assets

### Terra Trees (from Overworld2)
- `structures/vegetation/trees/` — 28 tree `.tesf` scripts (procedural + fixed)
  - Vanilla types: oak, birch, spruce, jungle, acacia, dark oak, mangrove, swamp
  - Custom types: eucalyptus, mesquite, palm, sakura, cherry
  - Rare forms: great azalea, azalea bush, windswept trees
- `structures/vegetation/crops/` — `propagule.tesf` (mangrove propagules)
- `structures/vegetation/vines/` — jungle/mangrove vine placements
- `structures/vegetation/mushrooms/` — tree-trunk mushroom disks
- `structures/vegetation/sculk/` — `spore_blossom.tesf`
- `structures/vegetation/bushes/` — `azalea_bush.tesf`
- `features/vegetation/trees/` — 22 `.yml` feature configs defining tree placement rules
- `features/vegetation/meta.yml` — shared `plantable-blocks` table

### Terra Decoration Pruned
- `structures/boulders/` — removed (vanilla handles boulders/ore deposits)
- `structures/deposits/` — removed (vanilla ore generation)
- `structures/misc/` — removed except `bee_nest.tesf` (referenced by trees)
- `structures/slabs/` — removed (vanilla slab placement)

## Remaining Work

- [ ] **Biome configs** — define 60+ biomes mapping vanilla types to Terra terrain + tree features
- [ ] **Noise settings** — match vanilla `noise_router`, `density_function`, and `noise` parameters
- [ ] **Carver configs** — Terra currently hard-codes carvers to no-op; needs NMS carver delegation or Terra carver configs
- [ ] **Pack manifest (`pack.yml`)** — wire generator type, biome provider, and `disable.structures: false`
- [ ] **Terrain samplers** — create density-function-based terrain matching vanilla `sloped_cheese`
- [ ] **Surface rules** — replace Terra's surface builder with vanilla-equivalent surface rules
- [ ] **Tree biome mapping** — assign which tree features go in which biomes (vanilla-equivalent placements)
