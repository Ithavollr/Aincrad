# Feature: Cross-World Portal Travel for Brain-Based Entities

## Status: INVESTIGATION COMPLETE — awaiting approval to implement

## Goal

Enable mob entities that use the `Brain` AI system (not the legacy `GoalSelector`) to travel
through nether portals cross-dimension, while **guaranteeing thread safety** under parallel
world ticking.

## Background — The Bug

When a Brain-based mob (e.g. Piglin) enters a nether portal in `world_nether`, the following
call chain fires during entity ticking on that world's dedicated thread:

```
ServerLevel.tickNonPassenger  (world_nether thread)
  → Entity.postTick
    → Entity.handlePortal
      → PortalProcessor.getPortalDestination
        → NetherPortalBlock.getPortalDestination   ← resolves target world (overworld)
          → NetherPortalBlock.getExitPortal
            → PortalForcer.findClosestPortalPosition
              → PoiAccess.findClosestPoiDataRecords
                → PoiManager.getOrLoad              ← BOOM: accesses overworld POI data
```

`PoiManager.getOrLoad` calls `TickThread.ensureTickThread(this.world, chunkX, chunkZ, …)`
where `this.world` is the **overworld**, but the current thread is the **nether's** tick worker.
The check fails, logs an ERROR, and throws `IllegalStateException`.

### Why this only affects non-player entities

- **Players**: `ServerPlayer.teleport()` already calls `ensureOnlyTickThread()` and is routed
  through the main tick thread, so the cross-world POI lookup never fires from a world-specific
  thread.
- **Non-player entities**: `Entity.postTick()` calls `handlePortal()` which calls
  `getPortalDestination()` inline on the current world's tick thread. The destination lookup
  reaches into the *other* world's POI data, violating thread isolation.

## Relevant Code & Files

### Portal lookup path (the crash site)
| File | Key Lines | Role |
|---|---|---|
| `Entity.java` | `postTick()` ~820, `handlePortal()` ~3273-3295 | Entry point, gates non-player portal logic |
| `PortalProcessor.java` | `getPortalDestination()` ~32 | Delegates to the portal block |
| `NetherPortalBlock.java` | `getPortalDestination()` ~146-182, `getExitPortal()` ~185-219 | Resolves target world, calls PortalForcer |
| `PortalForcer.java` | `findClosestPortalPosition()` ~47-83 | Calls PoiAccess on target world's PoiManager |
| `PoiAccess.java` | `findClosestPoiDataRecords()` ~200-373 | Iterates POI sections via `getOrLoad()` |
| `PoiManager.java` | `getOrLoad()` ~82-101 | Thread check: `ensureTickThread(this.world, …)` |

### Thread safety enforcement
| File | Key Lines | Role |
|---|---|---|
| `TickThread.java` | `isTickThreadFor(Level, int, int)` ~221-227 | Checks `currentlyTickingServerLevel == world` |
| `ServerPlayer.java` | `teleport()` ~1422 | `ensureOnlyTickThread(…)` — players already require main thread |

### Cross-dimension entity teleport
| File | Key Lines | Role |
|---|---|---|
| `Entity.java` | `teleportCrossDimension()` ~3822-3863 | Creates new entity via `getType().create()`, calls `restoreFrom()` |
| `Entity.java` | `restoreFrom()` ~3726-3739 | Serializes + deserializes NBT (preserves Brain state!) |

### Brain & POI memory system
| File | Role |
|---|---|
| `Brain.java` | Core Brain with `memories` map, `clearMemories()`, `stopAll()` |
| `MemoryModuleType.java` | Registry of all memory types |
| `Villager.java` | Defines `POI_MEMORIES` map, `releaseAllPois()`, `releasePoi()` |
| `AcquirePoi.java` | Behavior that acquires POI (job site, bed, etc.) |
| `NearestBedSensor.java` | Sensor that does POI lookup for beds |
| `SecondaryPoiSensor.java` | Sensor that scans blocks for secondary job sites |

### Brain-based entity types (all override `brainProvider()`)
Villager, Piglin, PiglinBrute, Hoglin, Zoglin, Allay, Axolotl, Frog, Tadpole, Goat,
Sniffer, Camel, Armadillo, Breeze, Creaking

## Thread Safety Analysis

There are **three** cross-world thread safety issues to solve:

### Issue 1: Portal destination POI lookup (the crash)
- **Where**: `PortalForcer.findClosestPortalPosition()` → `PoiManager.getOrLoad()`
- **Problem**: Nether tick thread reads overworld POI data
- **Severity**: Hard throw, blocks entity from portalling

### Issue 2: Villager `releasePoi()` during dimension change
- **Where**: `Villager.releasePoi()` at line ~690-706
- **Problem**: After crossing dimensions, if `releaseAllPois()` is called, it resolves
  `globalPos.dimension()` to the *old* world and accesses that world's PoiManager — another
  cross-world POI access from the wrong thread.
- **Severity**: Would hard throw if triggered

### Issue 3: Brain state carries stale cross-world references after teleport
- **Where**: `Entity.restoreFrom()` → `load()` deserializes the Brain from NBT
- **Problem**: `GlobalPos`-typed memories (HOME, JOB_SITE, POTENTIAL_JOB_SITE, MEETING_POINT,
  SECONDARY_JOB_SITE, HIDING_PLACE, DOORS_TO_CLOSE) still reference the old world's dimension
  key and block positions. Sensors and behaviors that act on these stale memories will
  pathfind to invalid locations or try cross-world POI operations.
- **Severity**: Logical errors, potential thread safety violations on next tick

## Proposed Implementation

### Phase 1: Fix the portal destination lookup (Issue 1) — CRITICAL

**Approach: Defer cross-world portal lookup to the main tick thread.**

In `Entity.handlePortal()` (~3273), when the entity is not a `ServerPlayer` and
`getPortalDestination()` would resolve to a different dimension:

1. Instead of calling `getPortalDestination()` inline, schedule the portal lookup + teleport
   as a task on the main server thread (via `MinecraftServer.execute()`).
2. The main tick thread is a plain `TickThread` (not a `ServerLevelTickThread`), so all
   `isTickThreadFor(world, …)` checks pass for any world.
3. The entity should be marked as "pending portal" so it isn't double-processed.

**Alternative considered**: Relax the thread check in `PoiManager.getOrLoad` for cross-world
reads. **Rejected** — POI data structures (`PoiChunk`, chunk holder maps) are not designed for
concurrent reads from multiple world threads. Even read-only access could race with load/unload
operations happening on the target world's thread.

### Phase 2: Wipe Brain POI memories on cross-dimension teleport (Issues 2 & 3)

**Approach: Clear all location-bound memories before the entity is recreated in the new world.**

In `Entity.teleportCrossDimension()`, after `restoreFrom()` but before `addDuringTeleport()`:

1. If the new entity is a `LivingEntity` with a Brain, call `brain.clearMemories()`.
2. Also call `brain.stopAll()` to halt any running behaviors that may hold references.
3. For Villagers specifically, call `releaseAllPois()` **on the old entity before
   `removeAfterChangingDimensions()`** — but this requires being on the correct thread for the
   old world's POI. Since Phase 1 ensures we're on the main thread, this is safe.

**Memories that reference world-specific locations (`GlobalPos` or `BlockPos`):**
- `HOME` — `GlobalPos` — Villager bed location
- `JOB_SITE` — `GlobalPos` — Villager workstation
- `POTENTIAL_JOB_SITE` — `GlobalPos` — unclaimed workstation
- `MEETING_POINT` — `GlobalPos` — bell location
- `SECONDARY_JOB_SITE` — `List<GlobalPos>` — composters, etc.
- `HIDING_PLACE` — `GlobalPos` — raid hiding spot
- `NEAREST_BED` — `BlockPos` — baby villager bed sensor
- `WALK_TARGET` — `WalkTarget` — contains target position
- `LOOK_TARGET` — `PositionTracker` — gaze target
- `PATH` — pathfinding data (world-bound)
- `DOORS_TO_CLOSE` — `Set<GlobalPos>` — door positions
- `INTERACTABLE_DOORS` — `List<GlobalPos>` — door positions

`clearMemories()` erases all of these in one call. The Brain's sensors and behaviors will
naturally repopulate them on subsequent ticks in the new world.

### Phase 3: Release old-world POI tickets before teleport (Villager-specific)

For Villagers, POI "tickets" (occupancy slots) must be released in the old world so other
villagers can claim those job sites / beds. This must happen **on the main thread** (which has
access to all worlds' POI) before the old entity is removed.

The sequence:
1. Main thread: `villager.releaseAllPois()` — frees old-world POI tickets
2. Main thread: `villager.getBrain().clearMemories()` — wipes stale references
3. Main thread: proceed with `teleportCrossDimension()` as normal

## Accepted Trade-offs

- **Villagers lose their profession** — job site POI released, memory cleared. They will pick
  up a new profession if an unclaimed workstation exists in the new world. This is acceptable.
- **Villagers lose their bed** — HOME memory cleared. They will claim a new bed.
- **Villagers lose gossip targets** — entity references don't survive cross-dimension.
- **Brief main-thread task** — the portal lookup + teleport runs on the main thread, adding a
  small scheduling delay. This only affects the frame where the entity completes its portal
  timer; subsequent ticks are back on world threads.
- **Piglins zombify in the overworld anyway** — Piglins that cross to the overworld will start
  zombification conversion (vanilla behavior in `AbstractPiglin`). Brain wipe is still needed
  to prevent stale memory issues during the conversion window.

## Open Risks & Questions

1. **Event ordering with plugins**: CraftBukkit fires `EntityPortalEvent` and
   `EntityTeleportEvent` during the portal path. Moving this to the main thread changes the
   thread these events fire on. Plugin compatibility risk — but plugins shouldn't be relying
   on world-tick-thread identity.

2. **Race window**: Between the entity's portal timer completing (on world thread) and the
   main-thread task executing, the entity continues ticking. It could die, be unloaded, or
   leave the portal. The main-thread task must re-validate entity state (alive, still in
   portal, not removed) before proceeding.

3. **Passenger chains**: `teleportCrossDimension()` recursively teleports passengers. If a
   passenger also has a Brain, its memories must also be cleared. The proposed `clearMemories()`
   call on the new entity after `restoreFrom()` handles this since each passenger goes through
   `teleportCrossDimension()` independently.

4. **Villager trade data**: Trades are stored separately from Brain and survive `restoreFrom()`.
   Profession reset only happens because the job site POI is released — if the villager has
   locked trades (level 2+), vanilla logic prevents profession loss. Need to verify this is
   still true after memory wipe.

5. **Warden / Breeze edge cases**: These Brain-based entities are unlikely to enter portals
   (Warden is underground, Breeze is in trial chambers), but the code path would apply to them
   too. `clearMemories()` is safe for all Brain-based entities.

6. **`PoiManager.release()` thread safety during Phase 3**: `releasePoi()` calls
   `poiManager.getType()` and `poiManager.release()` on the old world's PoiManager. Since
   Phase 1 ensures we're on the main tick thread, and main tick thread passes
   `isTickThreadFor(anyWorld)`, this is safe. However, if the old world is simultaneously
   ticking on its own thread and accessing the same POI chunk, there could be a data race on
   the POI section's occupancy counter. **This needs careful review** — it may be necessary to
   skip the `release()` call entirely and let the POI ticket expire or be cleaned up by the
   old world's own tick, at the cost of a temporarily "occupied" POI slot.

7. **Scope limitation**: Only Brain-based entities are targeted. Goal-based mobs (Zombies,
   Skeletons, Creepers, etc.) that wander into portals will continue to be blocked from
   cross-world travel. This is intentional per the feature request — goals are deprecated and
   we only support the newer system.
