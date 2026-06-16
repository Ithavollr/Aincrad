# Structure Spawn Override Replacement

Replace vanilla "indestructible" bounding-box structure spawning with trial spawners.
Players can break trial spawners to permanently disable spawning at a structure.

## Architecture

**Two systems working together:**

1. **Bounding-box spawn overrides (existing)** — converted to suppression-only (empty spawn lists).
   Prevents natural biome mobs from spawning inside structures, deconflicting with global mob caps.
2. **Trial spawners (new)** — placed inside structures during worldgen.
   Handle all positive mob spawning. Breakable (hardness 50, no drops), support ominous upgrades via Bad Omen.

## Server Code Changes

### 1. No-op positive spawn overrides in `ChunkGenerator.getMobsAt()`

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/chunk/ChunkGenerator.java` (line ~498-511)

Current behavior: If a structure has a `spawn_overrides` entry for a mob category, return that override's spawn list (which may contain mobs or be empty).

New behavior: Only return the override if its spawn list is **empty** (suppression). If the override has a non-empty spawn list, skip it and fall through to biome spawns.

```java
// Aincrad start - structure spawns use trial spawners now; only apply suppression overrides
if (structureSpawnOverride != null && structureSpawnOverride.spawns().isEmpty()) {
// Aincrad end
```

This single change disables all positive bounding-box spawning server-wide while preserving suppression (Ancient City, Trial Chambers, and our converted structures).

### 2. Remove line-of-sight requirement from trial spawner

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/block/entity/trialspawner/TrialSpawner.java` (line ~205)

Remove/bypass the `inLineOfSight()` check in `spawnMob()` so spawners work through walls, water, and terrain. Required for Ocean Monuments (underwater, enclosed rooms) and Nether Fortress (corridor walls).

```java
// Aincrad start - allow trial spawners to spawn through walls for structure use
// if (!inLineOfSight(level, pos.getCenter(), vec3)) {
//     return Optional.empty();
// }
// Aincrad end
```

### 3. Remove fortress fast-path in `NaturalSpawner.mobsAt()`

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/NaturalSpawner.java` (line ~429-435)

Current `mobsAt()` has a hardcoded fast-path: if the spawn position is inside a Nether Fortress and standing on nether bricks, it returns `FORTRESS_ENEMIES` directly, bypassing `getMobsAt()` entirely.

Remove this fast-path. The fortress JSON already has `spawn_overrides` for monster category — once converted to suppression-only, `getMobsAt()` handles suppression correctly. Trial spawners handle positive spawning.

```java
// Aincrad start - fortress spawning handled by trial spawners + suppression overrides
private static WeightedRandomList<MobSpawnSettings.SpawnerData> mobsAt(
    ServerLevel level, StructureManager structureManager, ChunkGenerator generator,
    MobCategory category, BlockPos pos, @Nullable Holder<Biome> biome
) {
    return generator.getMobsAt(biome != null ? biome : level.getBiome(pos), structureManager, category, pos);
}
// Aincrad end
```

### 4. Convert structure JSONs to suppression-only

**Files:** `aincrad-server/src/minecraft/resources/data/minecraft/worldgen/structure/`

For each structure with active spawns, replace mob entries with empty lists:

#### `pillager_outpost.json`
```json
"spawn_overrides": {
    "monster": { "bounding_box": "full", "spawns": [] }
}
```

#### `monument.json`
```json
"spawn_overrides": {
    "axolotls": { "bounding_box": "full", "spawns": [] },
    "monster": { "bounding_box": "full", "spawns": [] },
    "underground_water_creature": { "bounding_box": "full", "spawns": [] }
}
```

#### `fortress.json`
```json
"spawn_overrides": {
    "monster": { "bounding_box": "piece", "spawns": [] }
}
```

#### `swamp_hut.json`
```json
"spawn_overrides": {
    "creature": { "bounding_box": "piece", "spawns": [] },
    "monster": { "bounding_box": "piece", "spawns": [] }
}
```

### 5. Register new TrialSpawnerConfigs

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/block/entity/trialspawner/TrialSpawnerConfigs.java`

Register configs for each structure. Loot tables reference datapack-resolvable `ResourceKey<LootTable>` keys, so server operators can override rewards via datapack without touching server code.

#### Pillager Outpost
- **Mobs:** Pillager (weight 1)
- **spawnRange:** 14 (outpost tower + immediate surrounds)
- **simultaneousMobs:** 2 (vanilla spawns 1 at a time but continuously)
- **simultaneousMobsAddedPerPlayer:** 1
- **totalMobs:** 6
- **totalMobsAddedPerPlayer:** 2
- **ticksBetweenSpawn:** 40
- **targetCooldownLength:** 6000 (5 minutes)
- **Ominous:** Same mobs with equipment table, higher total/simultaneous counts
- **Loot:** Datapack loot table key `aincrad:spawner/pillager_outpost` (normal), `aincrad:spawner/pillager_outpost_ominous` (ominous drop table)

**Note:** Pillager Captains (patrol leaders with ominous banners) spawn naturally via `PatrollingMonster.finalizeSpawn()` with a 6% chance. This applies to trial spawner spawns (spawn reason `TRIAL_SPAWNER` is not excluded), so no special handling is needed.

#### Ocean Monument
- **Mobs:** Guardian (weight 1)
- **spawnRange:** 24 (monument is large, multiple spawners needed for full coverage)
- **simultaneousMobs:** 4 (vanilla maxCount per spawn)
- **simultaneousMobsAddedPerPlayer:** 1
- **totalMobs:** 8
- **totalMobsAddedPerPlayer:** 2
- **ticksBetweenSpawn:** 40
- **targetCooldownLength:** 6000
- **Ominous:** Guardian only (higher counts), NO Elder Guardians
- **Loot:** `aincrad:spawner/ocean_monument`, `aincrad:spawner/ocean_monument_ominous`

**Note:** Elder Guardians are one-time worldgen spawns (3 per monument: 1 in Penthouse, 2 in WingRooms). They do NOT respawn and are independent of the trial spawner system.

#### Nether Fortress
- **Mobs (weighted, replicating `FORTRESS_ENEMIES`):**
  - Blaze: weight 10, minCount 2, maxCount 3
  - Zombified Piglin: weight 5, minCount 4, maxCount 4
  - Wither Skeleton: weight 8, minCount 5, maxCount 5
  - Skeleton: weight 2, minCount 5, maxCount 5
  - Magma Cube: weight 3, minCount 4, maxCount 4
- **spawnRange:** 18 (corridors/rooms are moderately sized; multiple spawners cover the fortress)
- **simultaneousMobs:** 3
- **simultaneousMobsAddedPerPlayer:** 1
- **totalMobs:** 8
- **totalMobsAddedPerPlayer:** 2
- **ticksBetweenSpawn:** 40
- **targetCooldownLength:** 6000
- **Ominous:** Higher weights for Wither Skeleton + Blaze, equipment tables
- **Loot:** `aincrad:spawner/nether_fortress`, `aincrad:spawner/nether_fortress_ominous`

**Note:** The fortress's existing Blaze spawner block in MonsterThrone pieces (`NetherFortressPieces.java:1264`) remains unchanged — that's a regular `Blocks.SPAWNER`, not a trial spawner, and is independent.

#### Swamp Hut
- **Mobs:** Witch only (weight 1)
- **spawnRange:** 4 (small structure, single spawner)
- **simultaneousMobs:** 1
- **simultaneousMobsAddedPerPlayer:** 0.5
- **totalMobs:** 2
- **totalMobsAddedPerPlayer:** 1
- **ticksBetweenSpawn:** 80
- **targetCooldownLength:** 6000
- **Ominous:** Witches with Regeneration potion effect (same counts, no equipment)
- **Loot:** `aincrad:spawner/swamp_hut`, `aincrad:spawner/swamp_hut_ominous`

**Note:** A single black Cat is spawned as a one-time worldgen entity (like Elder Guardians) during `SwampHutPiece.postProcess()`. The trial spawner ONLY spawns Witches. For ominous mode, witches should spawn with a Regeneration potion effect instead of increased spawn counts.

**Implementation:** Add the Regeneration effect directly in the ominous spawn data NBT for witches (e.g., in the `entity` tag: `{"ActiveEffects":[{"Id":10,"Amplifier":0,"Duration":600}]}`). This requires no code changes — the NBT is passed directly to the spawned entity.

### 6. Register datapack loot tables (empty defaults)

**Directory:** `aincrad-server/src/minecraft/resources/data/aincrad/loot_table/spawner/`

Create empty loot tables for each structure spawner (normal + ominous). These exist as extension points — server operators can override them via datapack to add rewards.

- `aincrad:spawner/pillager_outpost`
- `aincrad:spawner/pillager_outpost_ominous`
- `aincrad:spawner/ocean_monument`
- `aincrad:spawner/ocean_monument_ominous`
- `aincrad:spawner/nether_fortress`
- `aincrad:spawner/nether_fortress_ominous`
- `aincrad:spawner/swamp_hut`
- `aincrad:spawner/swamp_hut_ominous`

### 7. Bump `MAX_MOB_TRACKING_DISTANCE`

**File:** `TrialSpawner.java` (line ~52)

Current value is 47 blocks. With `spawnRange` up to 24 and `requiredPlayerRange` up to 48, tracked mobs can easily exceed this. Increase to ~96 to prevent premature untracking.

## Spawner Placement

### NBT-based structures (user handles manually)

**Pillager Outpost** — Jigsaw structure with `.nbt` template files. User edits templates to place trial spawner blocks with block entity NBT referencing the registered config keys.

Each placed spawner's block entity NBT should include:

- `normal_config`: Reference to the registered config key (e.g., `aincrad:pillager_outpost/normal`)
- `ominous_config`: Reference to the ominous variant (e.g., `aincrad:pillager_outpost/ominous`)
- `required_player_range`: 32
- `target_cooldown_length`: 6000

### Procedural structures (server code changes required)

The following structures are NOT NBT-based — they generate blocks via Java `postProcess()` methods. Trial spawners must be placed programmatically in their piece classes, similar to how `NetherFortressPieces.MonsterThrone` already places a Blaze spawner.

#### 8. Place trial spawner in Swamp Hut

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/structure/structures/SwampHutPiece.java`

The hut is 7×7×9 blocks. Place a single trial spawner inside the hut at a suitable position (e.g., local coords `3, 2, 4` — center of the floor). Use `placeBlock()` to set the block, then set the block entity NBT with the swamp hut config.

The existing code spawns a one-time Witch and Cat entity during `postProcess()` (lines 91-106). **Remove the Witch spawn** (lines 91-103) but **keep the Cat spawn** (lines 105-123, `spawnCat()` method) — the Cat should remain as a one-time worldgen entity. Modify the Cat spawn to ensure it spawns a black cat specifically.

**Placement pattern** (following the existing `placeCraftSpawner` pattern but for trial spawner):
```java
// Aincrad start - place trial spawner for structure spawning
BlockPos spawnerPos = this.getWorldPos(3, 2, 4);
if (box.isInside(spawnerPos)) {
    level.setBlock(spawnerPos, Blocks.TRIAL_SPAWNER.defaultBlockState(), 2);
    if (level.getBlockEntity(spawnerPos) instanceof TrialSpawnerBlockEntity trialSpawnerBlockEntity) {
        // Set config references and parameters
    }
}
// Aincrad end
```

Config: `aincrad:swamp_hut/normal`, `aincrad:swamp_hut/ominous`
`required_player_range`: 14
`target_cooldown_length`: 6000

#### 9. Place trial spawners in Ocean Monument

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/structure/structures/OceanMonumentPieces.java`

The monument is ~58×58×23 blocks, generated as a `MonumentBuilding` piece that internally creates room pieces. Trial spawners should be placed in the main building piece's `postProcess()` at multiple positions to cover the large interior.

Recommended: Place spawners in the `MonumentBuilding.postProcess()` at 4-6 strategic positions distributed across the monument's internal volume (e.g., center of each floor level, wing rooms). Exact positions to be determined during implementation by examining the generation code.

Config: `aincrad:ocean_monument/normal`, `aincrad:ocean_monument/ominous`
`required_player_range`: 48
`target_cooldown_length`: 6000

#### 10. Place trial spawners in Nether Fortress

**File:** `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/structure/structures/NetherFortressPieces.java`

The fortress is composed of many corridor/room pieces. Rather than modifying every piece type, target the major room/corridor piece types that represent key areas:

- **`BridgeCrossing`** (large room at corridor intersections) — place 1 spawner at center
- **`BridgeStraight`** (long corridors) — place 1 spawner at midpoint
- **`CastleEntrance`** (fortress entrance hall) — place 1 spawner
- **`CastleSmallCorridorPiece`** or similar corridor pieces — place spawners at intervals

Each piece's `postProcess()` gets a spawner placement block, following the same pattern as the existing Blaze spawner in `MonsterThrone`. The `hasPlacedSpawner`-style boolean flag pattern (already used in MonsterThrone) prevents duplicate placement across chunk boundaries.

**Note:** The existing Blaze spawner in `MonsterThrone` (regular `Blocks.SPAWNER`) remains unchanged — it's an independent feature.

Config: `aincrad:nether_fortress/normal`, `aincrad:nether_fortress/ominous`
`required_player_range`: 48
`target_cooldown_length`: 6000

### Helper method for trial spawner placement

To avoid code duplication across piece classes, add a helper method to `StructurePiece` (or a utility class):

```java
// Aincrad start - trial spawner placement helper
protected void placeTrialSpawner(
    WorldGenLevel level, BlockPos pos, BoundingBox box,
    ResourceKey<TrialSpawnerConfig> normalConfig,
    ResourceKey<TrialSpawnerConfig> ominousConfig,
    int requiredPlayerRange, int targetCooldownLength
) {
    if (box.isInside(pos)) {
        level.setBlock(pos, Blocks.TRIAL_SPAWNER.defaultBlockState(), 2);
        if (level.getBlockEntity(pos) instanceof TrialSpawnerBlockEntity blockEntity) {
            TrialSpawner spawner = blockEntity.getTrialSpawner();
            spawner.normalConfig = level.registryAccess()
                .lookupOrThrow(Registries.TRIAL_SPAWNER_CONFIG).get(normalConfig).orElseThrow();
            spawner.ominousConfig = level.registryAccess()
                .lookupOrThrow(Registries.TRIAL_SPAWNER_CONFIG).get(ominousConfig).orElseThrow();
            spawner.requiredPlayerRange = requiredPlayerRange;
            spawner.targetCooldownLength = targetCooldownLength;
            blockEntity.setChanged();
        }
    }
}
// Aincrad end
```

## Per-spawner `requiredPlayerRange` recommendations

| Structure | `requiredPlayerRange` | Placement method | Spawner count                    |
|-----------|-----------------------|------------------|----------------------------------|
| Pillager Outpost | 32 | User edits NBT templates | 1                                |
| Ocean Monument | 48 | Java `postProcess()` | 4 (one in each corner if we can) |
| Nether Fortress | 48 | Java `postProcess()` | 1 per major piece                |
| Swamp Hut | 14 | Java `postProcess()` | 1                                |

## Implementation Order

1. Register `TrialSpawnerConfig`s and loot table keys
2. Create empty loot table JSON files
3. Convert structure JSONs to suppression-only
4. No-op positive spawn overrides in `ChunkGenerator.getMobsAt()`
5. Remove fortress fast-path in `NaturalSpawner.mobsAt()`
6. Remove line-of-sight check in `TrialSpawner.spawnMob()`
7. Bump `MAX_MOB_TRACKING_DISTANCE` in `TrialSpawner`
8. Add `placeTrialSpawner()` helper method
9. Add trial spawner placement to `SwampHutPiece.postProcess()`
10. Add trial spawner placement to `OceanMonumentPieces.MonumentBuilding.postProcess()`
11. Add trial spawner placement to Nether Fortress piece `postProcess()` methods
12. Test: verify structures suppress natural spawns but don't spawn via overrides
13. User places trial spawners in Pillager Outpost NBT templates
14. Test: verify trial spawners activate, spawn correct mobs, respect cooldown, support ominous
