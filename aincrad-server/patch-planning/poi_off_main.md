# POI Access Off the Owning World Thread

## Problem
POI data (`PoiSection` records: plain `Short2ObjectOpenHashMap` / `HashMap`) has no locking. The only guard is
Moonrise's `TickThread.ensureTickThread(world, chunkX, chunkZ, ...)` in `PoiManager.getOrLoad`, which always
hard-throws. Parallel world ticking narrowed that check from "any tick thread" to "the thread currently ticking
this world" (`TickThread.isTickThreadFor`), so any POI access from another thread or another world kills the
caller. Two classes of failure:

1. **Off main (shutdown race).** `ServerLevel.onBlockStateChange` defers POI writes via `getServer().execute(...)`.
   Once `MinecraftServer.stopped` is set, `scheduleExecutables()` is false and `execute()` runs the task inline on
   the caller. Worldgen workers keep generating until each world's chunk system halts in
   `ChunkHolderManager.close()`, so a structure placing a POI block during shutdown throws on
   `Paper Common Worker` and fails the chunk (`ChunkTaskScheduler.unrecoverableChunkSystemFailure`).
   Any `/stop` during pregen or exploration can hit it; AutoStop hits it on fresh worlds.
2. **Wrong world.** Villager brain memories hold `GlobalPos` (HOME, JOB_SITE, POTENTIAL_JOB_SITE, MEETING_POINT).
   Callers resolve `server.getLevel(globalPos.dimension())` and touch that world's `PoiManager` from the current
   world's tick thread: `Villager.releasePoi` (reached from `SetWalkTargetFromBlockMemory` x3, `releaseAllPois`),
   `GoToPotentialJobSite.stop`. `AssignProfessionFromJobSite` and `ValidateNearbyPoi` already guard on dimension.

## Current ownership (1.21.4)
- During a world's tick: that world's `ServerLevelTickThread`.
- Between ticks: the main `Server thread` (a plain `TickThread`), which runs queued tasks in
  `waitUntilNextTick` → `runAllTasks` while every world thread is idle (`tickChildren` blocks on their futures).
- The main thread passes `isTickThreadFor` for **any** world, so its POI access is never checked.

## Interim patch (`POI-Off-Main.patch`)
Fixes class 1 only; no change to normal runtime behaviour.
- `ServerLevel`: `poiUpdatesAfterStop` queue; `executePoiUpdate(task)` holds off-thread updates after stop,
  otherwise `getServer().execute(task)`. Both `onBlockStateChange` lambdas use it.
- `ChunkHolderManager.close()`: after the halt, before `saveAllChunks`, drains the queue on the main thread and
  logs `Applied N POI updates queued after stop`.
- Known gap: if the 60s halt times out, updates queued after the drain are lost.

## Full fix (after the 1.21.11 migration)
Principle: a world's POI data is owned by that world's `tickExecutor`, always. Re-verify every file/line below
against 1.21.11 before starting.

### Stage A — route to the owner
1. `ServerLevel.executePoiUpdate`: on this world's own tick thread run inline (restores vanilla timing: vanilla
   `execute()` runs same-thread tasks immediately); otherwise `this.tickExecutor.execute(task)`.
   Remove `poiUpdatesAfterStop` and the drain in `close()`.
2. Cross-world POI access (`Villager.releasePoi`, `GoToPotentialJobSite.stop`): if the target level is not the
   current level, submit the release to the **target** level's `tickExecutor`.
3. `ServerLevelTickExecutorThreadFactory`: set `currentlyTickingServerLevel` when the thread is created (the thread
   serves exactly one world). Today it is only set inside the tick job, so a task arriving before a world's first
   tick fails the owner check.
4. `ChunkHolderManager.close()`: after the halt, barrier on `world.tickExecutor.submit(() -> {}).get()` before
   `saveAllChunks`, so the world thread is idle while the main thread serializes POI data.
5. Unloaded worlds: `removeLevel` shuts the executor down; `execute()` then throws `RejectedExecutionException`.
   Drop the update in that case (the world is gone).

### Stage B — find the remaining race pieces
Moving writes onto world threads is only safe once nothing on the main thread touches a world's POI between ticks.
The current check cannot find those accesses (main thread always passes). Add a POI-specific owner check in
`PoiManager.getOrLoad` / `getOrCreate` that also rejects the main thread, then run until nothing throws. Known
candidates: tasks the main thread runs from its queue, commands (`LocateCommand` calls `getPoiManager()`; which
thread runs commands under parallel world ticking is unverified), plugin sync tasks. Move each onto the owning
world's executor.

## Verification
- Class 1: wipe worlds, `runServerTest` (AutoStop fires ~5s after Done, during spawn-region generation). No
  `failed main thread check` / `Chunk system error`; the drain log line appears when the race hits.
- Class 2: villager with a job site travels through a portal, then releases or loses the site. No thread throw.
- Stage B: strict check enabled, play-test villages, raids, portals, `/locate poi`. Zero throws.
