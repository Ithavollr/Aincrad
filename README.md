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

## SETUP
### Getting Started (new machines)
1. Clone repo
2. `./gradlew applyPatches` from root

## REPO SYNC
1. `git checkout main` - switch to the main branch, that tracks Paper
2. via Github UI, PR all new changes into main
3. Make a list of any code that should make it into Aincrad
4. `git checkout seed`
5. `git cherry-pick <new_paper_goodness_commit_hash>`
6. `gradlew applyPatches` - this makes sure your patch/source files are in sync. Forgetting this step will lead to errors the next time you `rebuildPatches`
7. Push the changes back to origin, all done!

> [!NOTE]
> for mistakes,  
> Reset all non-committed local changes: `git reset --hard`  
> Reset branch to remote state: `git reset --hard origin/seed`  
> Reset to specific commit: `git reset --hard <commit-hash>`,  
>   then `git push origin seed --force`

> [!TIP]
> Current sync status tracked [here](paper-server/README.md)

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
