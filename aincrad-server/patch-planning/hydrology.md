# Hydrology (MVP1): streams, rivers and lakes

## Problem
Aincrad's rivers are valley dips in the terrain noise (`TerrainProvider.overworldOffset`, the `ridges_folded` valleys)
filled with water at sea level (`NoiseBasedChunkGenerator.createFluidPicker`). They do not start in mountains, do not
flow downhill, and cannot sit above sea level.

## Fix
Per-plate river tracing after the Plate-Local River Generation Algorithm (`references/plate-local-river-generation`,
`flat_infinite/src/rivers.cpp`), applied during `NOISE`:

- **Sections:** borders follow drainage divides (ridges) and coastlines found in the planning heightmap, so rivers never
  need to cross one (PLRGA gets this for free because its terrain is built from its plates; ours comes from Ferma, so
  the sections are found from the terrain instead). Each section is computed alone.
- **Size cap: 1024 blocks** (Streams Reflowing's tested default). Where the terrain does not divide an area, for example
  a Ferma configuration with no mountains, the cap draws the border, so a section can never grow without bound.
  Seeded from the world (`RandomState.getOrCreateRandomFactory`).
- **Planning heights:** `preliminarySurfaceLevel` every 12 blocks (Streams' gridSpacing default; Ferma-patched router, same estimate as the seed
  previews). Needs no finished chunks, so no chunk step is held back.
- **Tracing:** sources at the highest cells (minimum spacing), steepest downhill step, join an existing river as a
  tributary, end at sea level or at a local minimum (becomes a lake). No crossings.
- **Water height:** drops steadily from source to mouth. Width from Strahler order, sized with Streams' width defaults
  (`baseWidth` 0.4, `maxWidth` 16.0, `widthScale` 1.0; how they map to blocks is read in step 3).
- **Per chunk:** each column takes its distance to the nearest segment and gets a bed Y and a water Y: the channel is
  carved and water filled at that column's own surface, at any elevation.
- **Cache:** finished plates per `RandomState` (one per world and seed).

## Build order
1. **Aquifer wall fix** (done, see Files to Modify).
2. **Heightmap, sections and lakes:** planning grid, sections with the 1024 cap, depression fill per section. Basins
   filled above the raw ground become lakes with a flat water level; basins under 512 blocks of area or under
   3 blocks deep (less 1 for estimated heights) are dropped, all Streams defaults. Water placed
   right after the terrain fill, on the real ground (`OCEAN_FLOOR_WG`) up to the lake level, as Streams does.
3. **Rivers:** tracing on the filled grid, ending at sea or in the lakes from step 2.

---

## Files to Create

### `aincrad-server/src/minecraft/java/org/evlis/hydrology/NaturalRivers.java`
- `public` (called from `net.minecraft`). Same `org.evlis` root as paper-server's Ridables classes; no module-info,
  so no split-package conflict.
- Pure-Java core: plate lattice, height grid, tracing, Strahler order, lakes, segment lookup bucketed by area.
- MC-facing:
  - `fillLakes(chunk, randomState, seaLevel)`: after the terrain fill, air from each lake column's top solid block
    (`OCEAN_FLOOR_WG`) up to the lake level becomes water. Same floor source as Streams (`RiverCarverFeature.java:276`),
    so caves under a lake stay dry and no floor estimate or slack is needed.
  - `inChannel(x, z)`: whether a column is inside a channel or lake.
- Self-check: a synthetic cone of terrain must produce rivers that only flow downhill and end at sea or in a lake.

---

## Files to Modify

### `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/Aquifer.java` (done, playtested)
- `computeSubstance`, the first barrier check (line 251, cells 1 vs 2) only:
  ```java
  if (substance + d1 > 0.0 && !(aquiferStatus.fluidType().is(Blocks.WATER) && aquiferStatus2.fluidType().is(Blocks.WATER))) { // Aincrad - hydrology: water joins, no barrier
  ```
- Removes the one-block wall between two water bodies, or water and a carved cave, along the boundary between the two
  nearest aquifer cells. Dry cave cells are water-typed (level `WAY_BELOW_MIN_Y`), so they count as water here.
- **Keep the other two checks (lines 259 and 268, cells 1 vs 3 and 2 vs 3) unchanged.** They keep the corners where
  three cells meet. With all three changed, whole faces open: the queued fluid update (`markPosForPostprocessing`)
  starts flow, and water source conversion (`FlowingFluid.getNewLiquid`, two neighbouring sources) chains through the
  cave and fills it with source blocks. With only this check, breaks stay about two blocks wide and no sources build up.

### `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/NoiseBasedChunkGenerator.java`
- `fillFromNoise`: call `NaturalRivers.fillLakes` right after `doFill`, inside the same `try`, while the chunk's sections
  are still acquired.

### `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/carver/WorldCarver.java`
- `carveBlock` (line 124): return `false` when `NaturalRivers.inChannel(x, z)`, so caves do not drain rivers.

---

## Not in MVP1
- River biomes.
- Rivers in the seed previews.
- Rivers crossing plate borders.
- Flow direction, banks, vegetation.

## Unverified, check first
- Cost of one tile (87 x 87 height samples). Not measured yet.
