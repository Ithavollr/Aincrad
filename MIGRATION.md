# Aincrad → Purpur-Style Patch System Migration Plan

## Executive Summary

**Current State:** Aincrad is a hard fork of Paper using `paperweight-core`, with all patches (Paper + custom) stored locally.
**Target State:** Aincrad uses `paperweight-patcher` to track Paper as upstream, with custom patches layered on top.

**Benefits After Migration:**
- Update Paper by changing a single commit hash in `gradle.properties`
- No more cherry-picking commits from Paper
- Clean separation between Paper patches and Aincrad custom patches
- Smaller repository (Paper patches live in upstream, not your repo)

---

## Architecture Comparison

### Current: Hard Fork (paperweight-core)

```
Aincrad Repo
├── paper-api/            ← Paper API with patches applied directly
├── paper-server/         ← Paper server with patches applied directly
│   ├── patches/
│   │   ├── sources/      ← 838 file patches (Paper + Aincrad mixed)
│   │   ├── features/     ← 37 feature patches (Paper + Aincrad mixed)
│   │   └── resources/    ← 6 resource patches
│   └── src/minecraft/    ← Decompiled MC + patched sources
```

**Problems:**
- All Paper patches are in your repo (800+ patches)
- To update Paper, you must cherry-pick individual commits
- Custom patches (0032-0037) are mixed with Paper patches (0001-0031)

### Target: Fork with Upstream (paperweight-patcher)

```
Aincrad Repo
├── paper-api/            ← Generated: Paper API after Paper's patches
├── paper-server/         ← Generated: Paper server after Paper's patches
├── aincrad-api/          ← NEW: Your API patches on top of Paper
│   ├── build.gradle.kts.patch
│   └── paper-patches/
│       ├── files/        ← File patches to Paper API
│       └── features/     ← Feature patches to Paper API
├── aincrad-server/       ← NEW: Your server patches on top of Paper
│   ├── build.gradle.kts.patch
│   ├── paper-patches/    ← Patches to Paper server classes
│   │   ├── files/
│   │   └── features/
│   └── minecraft-patches/← Patches to Minecraft classes
│       ├── sources/
│       └── features/
└── gradle.properties
    └── paperCommit = abc123  ← Just change this to update Paper!
```

---

## Custom Patches to Preserve (Aincrad-Specific)

Based on analysis of `/home/freyja/code/Aincrad/paper-server/patches/features/`:

| Patch # | Name | Description | Migration Target |
|-----------|------|-------------|------------------|
| 0032 | Parallel World Ticking SP | Formatting fixes to moonrise | `aincrad-server/paper-patches/files/` |
| 0033 | Network Modifications | Debug logging changes to ServerConfigurationPacketListenerImpl | `aincrad-server/paper-patches/files/` |
| 0034 | Water World Modifications | Makes Shulkers aquatic | `aincrad-server/minecraft-patches/sources/net/minecraft/world/entity/monster/Shulker.java.patch` |
| 0035 | Remove End Dragon Battle | Removes dragon fight from The End | `aincrad-server/minecraft-patches/sources/net/minecraft/server/level/ServerLevel.java.patch` |
| 0036 | Giants AI | Gives Giants AI (goals, targets) | `aincrad-server/minecraft-patches/sources/net/minecraft/world/entity/monster/Giant.java.patch` |
| 0037 | Per-world Monster Limits | Entity counting per world | `aincrad-server/minecraft-patches/sources/net/minecraft/server/level/ServerLevel.java.patch` |

**Note:** Patches 0001-0031 are standard Paper patches and will come from upstream automatically.

---

## Migration Steps

### Phase 1: Repository Preparation

1. **Create backup branch**
   ```bash
   git checkout -b backup/hard-fork-archive
   git push origin backup/hard-fork-archive
   git checkout main
   ```

2. **Clean working directory**
   - Ensure no uncommitted changes
   - Run `./gradlew clean` to remove build artifacts

### Phase 2: Structural Changes

3. **Update `build.gradle.kts`**
   - Replace `io.papermc.paperweight.core` plugin with `io.papermc.paperweight.patcher`
   - Add `paperweight` block with upstream configuration
   - See reference: `Purpur/build.gradle.kts`

4. **Update `settings.gradle.kts`**
   - Change project name from `aincrad` to `aincrad` (keep)
   - Update module names from `paper-api`, `paper-server` to `aincrad-api`, `aincrad-server`
   - Keep paper-api and paper-server as generated directories

5. **Update `gradle.properties`**
   - Add `paperCommit = <current_paper_commit_hash>`
   - Keep existing mcVersion and version properties

6. **Create new module directories**
   ```
   mkdir -p aincrad-api/paper-patches/{files,features}
   mkdir -p aincrad-server/{paper-patches/{files,features},minecraft-patches/{sources,features}}
   mkdir -p aincrad-server/src/main/java
   ```

### Phase 3: Extract Custom Patches

7. **Extract and convert custom feature patches**

   For each custom patch (0032-0037), you need to:
   - Determine if it patches Minecraft classes or Paper classes
   - Convert to the appropriate patch location

   **Example: Patch 0036 (Giant AI)**
   - This patches `net/minecraft/world/entity/monster/Giant.java`
   - Belongs in: `aincrad-server/minecraft-patches/sources/net/minecraft/world/entity/monster/Giant.java.patch`
   - Format change needed: The patch header must reference the file correctly

8. **Reformat patches for paperweight-patcher**

   Current format (paperweight-core):
   ```patch
   --- a/net/minecraft/world/entity/monster/Giant.java
   +++ b/net/minecraft/world/entity/monster/Giant.java
   ```

   New format (paperweight-patcher):
   ```patch
   --- a/src/minecraft/java/net/minecraft/world/entity/monster/Giant.java
   +++ b/src/minecraft/java/net/minecraft/world/entity/monster/Giant.java
   ```

### Phase 4: Build Configuration Migration

9. **Create `aincrad-server/build.gradle.kts.patch`**
   - This is a patch to the generated `paper-server/build.gradle.kts`
   - Must:
     - Register `aincrad` fork
     - Configure source sets to include both `paper-server/src` and `aincrad-server/src`
     - Change dependency from `paper-api` to `aincrad-api`
     - Update manifest attributes for branding

10. **Create `aincrad-api/build.gradle.kts.patch`**
    - Similar structure to server patch
    - Configure source sets
    - Update javadoc paths

### Phase 5: Custom Source Code Migration

11. **Move custom source files**
    - Any custom classes in `paper-server/src/main/java` → `aincrad-server/src/main/java`
    - Update package declarations from `io.papermc.paper` to your own (e.g., `io.aincrad`)

12. **Configuration classes**
    - If you have custom config classes (like Purpur's `PurpurConfig`), place in `aincrad-server/src/main/java`

### Phase 6: Testing & Validation

13. **First build attempt**
    ```bash
    ./gradlew applyAllPatches
    ```
    - This will download Paper at the specified commit
    - Apply Paper's patches
    - Then apply your custom patches on top

14. **Fix any patch failures**
    - If patches fail to apply, enter the paper-server directory
    - Resolve conflicts manually
    - Run `./gradlew rebuildMinecraftPatches` or `./gradlew rebuildPaperPatches`

15. **Verify build**
    ```bash
    ./gradlew build
    ```

### Phase 7: Cleanup

16. **Remove old patch directories**
    - After successful build, remove `paper-server/patches/` (these are now from upstream)
    - Paper patches now come from the Paper commit, not your repo

17. **Update documentation**
    - Update `CONTRIBUTING.md` with new patch workflow
    - Document how to update upstream: just change `paperCommit` in gradle.properties!

---

## Key File References

### Purpur's Implementation (for reference)

| File | Purpose |
|------|---------|
| `Purpur/build.gradle.kts` | Root build with `paperweight-patcher` plugin |
| `Purpur/settings.gradle.kts` | Project structure with purpur-api/purpur-server |
| `Purpur/gradle.properties` | Contains `paperCommit` hash |
| `Purpur/purpur-server/build.gradle.kts.patch` | Patches to paper-server build |
| `Purpur/purpur-api/build.gradle.kts.patch` | Patches to paper-api build |
| `Purpur/purpur-server/paper-patches/` | Patches to Paper server classes |
| `Purpur/purpur-server/minecraft-patches/` | Patches to Minecraft classes |
| `Purpur/purpur-api/paper-patches/` | Patches to Paper API classes |

### Aincrad Files to Modify

| File | Action |
|------|--------|
| `build.gradle.kts` | Replace plugin, add upstream block |
| `settings.gradle.kts` | Update project names, add aincrad-api/server |
| `gradle.properties` | Add `paperCommit` |
| `paper-server/build.gradle.kts` | Will be generated from upstream |
| `paper-api/build.gradle.kts` | Will be generated from upstream |
| `aincrad-server/build.gradle.kts.patch` | NEW - patches to paper-server build |
| `aincrad-api/build.gradle.kts.patch` | NEW - patches to paper-api build |

---

## Patch Conversion Reference

### File Patch Conversion

**From:** `paper-server/patches/sources/net/minecraft/Util.java.patch`

```patch
--- a/net/minecraft/Util.java
+++ b/net/minecraft/Util.java
```

**To:** `aincrad-server/minecraft-patches/sources/net/minecraft/Util.java.patch`

```patch
--- a/src/minecraft/java/net/minecraft/Util.java
+++ b/src/minecraft/java/net/minecraft/Util.java
```

### Feature Patch Conversion

**From:** `paper-server/patches/features/XXXX-Name.patch`
(multiple files in one patch)

**To:** `aincrad-server/minecraft-patches/features/XXXX-Name.patch`
or `aincrad-server/paper-patches/features/XXXX-Name.patch`

Depending on whether the patch touches:
- Minecraft classes → `minecraft-patches/features/`
- Paper classes → `paper-patches/features/`

---

## Risks & Mitigation

| Risk | Mitigation |
|------|------------|
| Custom patches don't apply cleanly | Keep backup branch; resolve conflicts one patch at a time |
| Missing Paper patches after migration | Ensure `paperCommit` points to a commit that includes all needed Paper features |
| Build breaks after restructuring | Test `./gradlew applyAllPatches` before committing changes |
| Lost custom source files | Move (don't delete) src files; verify with `git status` |
| Patch format confusion | Test one patch conversion thoroughly before doing all |

---

## Post-Migration Workflow

### Updating Paper (The Big Win!)

Before (hard fork):
```bash
# Cherry-pick individual commits from Paper
git remote add paper https://github.com/PaperMC/Paper.git
git fetch paper
git cherry-pick abc123  # repeat for each commit
git cherry-pick def456
# Resolve conflicts for each...
```

After (paperweight-patcher):
```bash
# Edit gradle.properties
# Change: paperCommit = oldhash
# To:     paperCommit = newhash
./gradlew applyAllPatches
# Done! Paper patches applied automatically.
# Only need to fix your custom patches if they conflict.
```

### Adding New Custom Patches

```bash
# 1. Apply all patches to work on code
./gradlew applyAllPatches

# 2. Make changes in the generated directories
cd paper-server/src/minecraft/java
# Edit files...

# 3. Fixup file patches (per-file changes)
./gradlew fixupMinecraftFilePatches

# 4. Or rebuild feature patches (multi-file changes)
git add . && git commit -m "My feature"
./gradlew rebuildMinecraftFeaturePatches
```

---

## Timeline Estimate

| Phase | Estimated Time |
|-------|----------------|
| 1-2: Preparation | 30 minutes |
| 3: Build config changes | 1-2 hours |
| 4-5: Patch extraction & conversion | 3-4 hours |
| 6: Testing & debugging | 2-3 hours |
| 7: Cleanup & docs | 1 hour |
| **Total** | **~8-12 hours** |

---

## Decision Checklist

Before starting migration, confirm:

- [ ] You have a clean git state (no uncommitted changes)
- [ ] You've created a backup branch
- [ ] You've identified the exact Paper commit you currently base on
- [ ] You've listed all custom patches that need to be preserved
- [ ] You understand that after migration, Paper patches won't be in your repo
- [ ] You're prepared to resolve patch conflicts during the transition

---

## Questions to Resolve Before Starting

1. **What Paper commit should be the initial upstream?**
   - Find by looking at `paper-server/patches/features/0001-*.patch` headers
   - Or check git history for "Upstream update" commits

2. **Should custom patches be combined or kept separate?**
   - Recommendation: Keep separate initially for easier debugging
   - Can combine later once system is stable

3. **What namespace for custom classes?**
   - Currently uses `io.papermc.paper` in patches
   - Consider: `io.aincrad` or keep Paper namespace

4. **Which patches modify the same files?**
   - Patch 0035 and 0037 both modify `ServerLevel.java`
   - May need to combine or carefully sequence these
