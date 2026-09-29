# Village Y-Delta Cap

## Problem
Villages can spawn on steep terrain (mountains, cliffs) where individual pieces are placed at wildly different heights. Villagers spawned on high pieces have no path down and fall to their deaths as soon as a player loads the chunk.

## Fix
Create a custom village-only jigsaw placement that rejects any piece whose terrain Y deviates more than `MAX_Y_DELTA` from the village center's ground Y.

**`MAX_Y_DELTA = 20`**

---

## Files to Create

### `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/structure/pools/VillageJigsawPlacement.java`
- Copy of `JigsawPlacement.java` with village Y-cap logic added.
- Wrap all changes in `// Aincrad start` / `// Aincrad end` comments.
- Key change in `Placer`:
  - Add `final int centerGroundY` field, set in constructor.
  - In `tryPlacingChildren`, after computing `i4` (candidate piece's Y), add before the collision check:
    ```java
    // Aincrad start - village Y-delta cap
    if (Math.abs(i4 - this.centerGroundY) > MAX_Y_DELTA) {
        continue;
    }
    // Aincrad end
    ```
  - `MAX_Y_DELTA` is a constant on the class: `private static final int MAX_Y_DELTA = 20;`
- `centerGroundY` is passed from `addPieces`, which reads it as:
  `startPiece.getBoundingBox().minY() + startPiece.getGroundLevelDelta()`

---

## Files to Modify

### `aincrad-server/src/minecraft/java/net/minecraft/world/level/levelgen/structure/structures/JigsawStructure.java`
- Add a `boolean useVillagePlacement` field with codec support (`Codec.BOOL.optionalFieldOf("use_village_placement", false)`).
- In `findGenerationPoint`, dispatch on this flag:
  ```java
  // Aincrad start - village Y-delta cap
  if (this.useVillagePlacement) {
      return VillageJigsawPlacement.addPieces(...);
  }
  // Aincrad end
  return JigsawPlacement.addPieces(...);
  ```

### Village structure data / `BuiltinStructures` registrations
- Set `use_village_placement: true` on all five village structures:
  `VILLAGE_PLAINS`, `VILLAGE_DESERT`, `VILLAGE_SAVANNA`, `VILLAGE_SNOWY`, `VILLAGE_TAIGA`

---

## Notes
- This does NOT modify `JigsawPlacement.java` — vanilla jigsaw structures (bastions, trial chambers, etc.) are unaffected.
- Pieces that exceed `MAX_Y_DELTA` are skipped entirely; the jigsaw fallback pool (typically `minecraft:empty`) handles the open connector, same as any other unresolvable jigsaw join.
