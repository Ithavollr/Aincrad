# Aincrad 🛡️ [![Discord](https://img.shields.io/discord/1211431882957267024.svg?label=&logo=discord&logoColor=ffffff&color=7389D8&labelColor=6A7EC2)](https://discord.gg/RnKzBfWh7j)
This is the custom server code used in the Minecraft world of Iðavöllr.

**Changes from PaperMC**
- [ ] Re-implement ALL old Aincrad patches.....  

| # | Filename | Original |
|---|----------|----------|
| 0001 | Parallel-World-Ticking-SP | 0032 |
| 0002 | Network-Modifications | 0033 |
| 0003 | Water-World-Modifications | 0034 |
| 0004 | Remove-End-Dragon-Battle | 0035 |
| 0005 | Giants-AI | 0036 |
| 0006 | Per-world-Monster-Limits | 0037 |
- [x] Edit max speeds so minecart > horse > ice boat (paddles only, sails should still be fast)
- [x] Implement Giants AI
- [ ] Add new source of Levitation Effect
- [x] Strongly suggest Villagers do not swim
- [x] Make Shulkers aquatic, fix shulker bullet water pathing
- [x] Make Chorus fruit aquatic
- [ ] Implement [Purpur](https://github.com/PurpurMC/Purpur) rideables
- [ ] Implement Villager Tasks (Armorer heals golems, Priest heals villagers)
- [x] Implement [Sparkly](https://github.com/SparklyPower/SparklyPaper) per-world ticking
- [ ] Integrate [Denizen](https://github.com/DenizenScript/Denizen) for _all_ entity goals & behaviours
  - [ ] Merge NMS/world/entity/ai/behavior
  - [ ] Merge NMS/world/entity/ai/goal
- [ ] Find out how Brain.java is leaking default-world POI's to serverLevelTickExecutors (so far detected in `SetWalkTargetFromBlockMemory` & `AssignProfessionFromJobSite`)
  - [ ] Re-implement [SmartBrainLib](https://github.com/Tslat/SmartBrainLib)
- [ ] Inspect and understand net/minecrarft/core/dispenser parallel ticking updates

## SETUP
### Getting Started (new machines)
1. Clone repo
2. `./gradlew applyAllPatches` from root

## REPO SYNC

Aincrad is a set of patches applied on top of Paper. The Paper upstream commit is tracked in `gradle.properties` as `paperCommit`.

### Updating Paper upstream

1. Update `paperCommit` in `gradle.properties` to the latest commit hash from Paper's version branch (e.g., `ver/1.21.4`)
2. Run `./gradlew applyAllPatches` to regenerate source directories with the new Paper base
3. If patch conflicts occur, resolve them in the source directories (`aincrad-server/src/minecraft/java/` or `paper-server/`)
4. Run `./gradlew rebuildPatches` to regenerate the patch files from your changes
5. Review and commit the updated patches

> [!NOTE]
> **Handling conflicts:**
> - After `applyAllPatches`, check for `.rej` files indicating failed patch hunks
> - Resolve conflicts manually in the source files, then run `rebuildPatches`
> - Some patches may need to be dropped entirely if Paper has superseded them

> [!TIP]
> **For mistakes:**  
> Reset all non-committed local changes: `git reset --hard`  
> Clean build artifacts: `./gradlew clean`  
> Regenerate everything: `./gradlew applyAllPatches`

## BUILD
1. Go to the gradle tasks -> bundling
2. `createMojmapBundlerJar`

## CHANGES

There are two working trees, each with its own patch set:

| Working tree | Patch dir | Gradle prefix | Contains |
|---|---|---|---|
| `aincrad-server/src/minecraft/java` | `aincrad-server/minecraft-patches/` | `rebuildMinecraft` / `fixupMinecraft` | Decompiled Minecraft classes (`net/minecraft/...`) |
| `paper-server/` | `aincrad-server/paper-patches/` | `rebuildPaperServer` / `fixupPaperServer` | Paper + moonrise + craftbukkit classes |

### Per-file changes (single-file patch, no commit needed)

**Minecraft classes:**
1. Edit file in `aincrad-server/src/minecraft/java/`
2. `./gradlew fixupMinecraftFilePatches` from root

**Paper/moonrise classes:**
1. Edit file in `paper-server/src/main/java/`
2. `./gradlew fixupPaperServerFilePatches` from root

### Feature changes (multi-file patch via git commit)

**Minecraft classes:**
1. Edit files in `aincrad-server/src/minecraft/java/`
2. `git add . && git commit -m "Your feature name"` inside `aincrad-server/src/minecraft/java/`
3. `./gradlew rebuildMinecraftFeaturePatches` from root

**Paper/moonrise classes:**
1. Edit files in `paper-server/src/main/java/`
2. `git add . && git commit -m "Your feature name"` inside `paper-server/`
3. `./gradlew rebuildPaperServerFeaturePatches` from root

### Fixup an existing feature patch

**Minecraft classes:**
1. Edit files in `aincrad-server/src/minecraft/java/`
2. `git log` inside that directory — find the target commit hash
3. `git commit -a --fixup <target_hash>` inside that directory
4. `git rebase -i --autosquash base` inside that directory
5. `./gradlew rebuildMinecraftFeaturePatches` from root

**Paper/moonrise classes:**
1. Edit files in `paper-server/src/main/java/`
2. `git log` inside `paper-server/` — find the target commit hash
3. `git commit -a --fixup <target_hash>` inside `paper-server/`
4. `git rebase -i --autosquash base` inside `paper-server/`
5. `./gradlew rebuildPaperServerFeaturePatches` from root

#### Add files for patching
find the file needed using "view source" or manually in the gradle cache, add the full classpath to `./build-data/dev-imports.txt`, run the Gradle task "applyPatches", and you should be able to find your new NMS file in the `./Paper-Server` dir.

#### VERY helpful guide to patching:
[PaperMC Contrib](https://github.com/PaperMC/Paper/blob/master/CONTRIBUTING.md)

> [!TIP]
> If IntelliJ Idea is still not resolving references,
> go to File -> Invalidate Caches, delete the .idea folder,
> and restart the program.
