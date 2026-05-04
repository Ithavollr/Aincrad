# Migration Plan: Hard Fork → paperweight-patcher (Purpur-style)

**Goal:** Build cleanly from Paper commit `6abf5f0b747db2d9d098d5d35f4859f68f20c867` (ver/1.21.4 branch) with zero custom patches, structured like Purpur.

**Starting point:** `backup/hard-fork-archive` branch.

---

## Steps

### 1. Create working branch
```bash
git checkout backup/hard-fork-archive
git checkout -b seed
```

### 2. Replace root `build.gradle.kts`
Replace the `paperweight-core` plugin block with `paperweight-patcher`. Reference: `Purpur/build.gradle.kts`.

Key changes:
- Plugin: `id("io.papermc.paperweight.patcher") version "2.0.0-beta.14"`
- Add `repositories { maven("https://jitpack.io") }` inside `subprojects`
- Add `paperweight { upstreams.paper { ref = providers.gradleProperty("paperCommit"); patchDir("paperApi") { ... } } }` block
- **No `patchFile` blocks** — `aincrad-server/build.gradle.kts` and `aincrad-api/build.gradle.kts` are committed directly, not generated from patches

### 3. Replace `settings.gradle.kts`
- Include `aincrad-api` and `aincrad-server` subprojects (not `paper-api`/`paper-server`)
- Reference: `Purpur/settings.gradle.kts`

### 4. Update `gradle.properties`
Add:
```properties
paperCommit = 6abf5f0b747db2d9d098d5d35f4859f68f20c867
```

### 5. Update `gradle/wrapper/gradle-wrapper.properties`
Set Gradle to `8.12` (required by paperweight-patcher 2.0.0-beta.14):
```
distributionUrl=https\://services.gradle.org/distributions/gradle-8.12-bin.zip
```

### 6. Create `aincrad-server/` and `aincrad-api/` module directories
```bash
mkdir -p aincrad-server/paper-patches/features
mkdir -p aincrad-server/paper-patches/files
mkdir -p aincrad-server/minecraft-patches/features
mkdir -p aincrad-server/minecraft-patches/sources
mkdir -p aincrad-server/minecraft-patches/resources
mkdir -p aincrad-api/paper-patches/features
mkdir -p aincrad-api/paper-patches/files
```
Leave all patch dirs **empty**.

### 7. Create `aincrad-server/build.gradle.kts`
Copy `paper-server/build.gradle.kts` and make these changes:
- Add `paperweight { forks.register("aincrad") { ... }; activeFork = aincrad }` block (reference: `Purpur/purpur-server/build.gradle.kts`)
- Add `sourceSets` block pointing to `../paper-server/src/main/java` etc.
- Change `implementation(project(":paper-api"))` → `implementation(project(":aincrad-api"))`
- Update manifest: `Implementation-Title`, `Specification-Title`, `Brand-Id`, `Brand-Name` to Aincrad values
- Pin any snapshot dependencies to their resolved timestamps (configurate, spark)

### 8. Create `aincrad-api/build.gradle.kts`
Copy `paper-api/build.gradle.kts` and make these changes:
- Fix `generatedApiPath` to point to `../paper-api/src/generated`
- Expand `sourceSets` to include `../paper-api/src/main/java` etc.
- Fix javadoc `inputs.dir` to `../paper-api/src/main/javadoc`

### 9. Update `.gitignore`
Add:
```
/paper-server
/paper-api
/aincrad-server/src/minecraft
*/build/
*.jar
*.rej
*.orig
```
Remove tracking of `paper-server/` and `paper-api/` if previously committed:
```bash
git rm -r --cached paper-server/ paper-api/ 2>/dev/null
```

### 10. Run and verify
```bash
./gradlew applyAllPatches --no-daemon
```
Expected: `BUILD SUCCESSFUL` with all `Applied 0 patches` lines.

---

## What NOT to do
- Do **not** create `build.gradle.kts.patch` files — just commit the files directly
- Do **not** put custom patches in `minecraft-patches/` — all Aincrad patches go in `paper-patches/features/` (applied to the combined working tree)
- Do **not** strip `index` lines from patches manually — use `--no-3way` when applying

---

# Migration Plan: Hard Fork → paperweight-patcher (Purpur-style) [OLD NOTES BELOW]

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

### Phase 1: Repository Preparation - COMPLETE
Backup branch created and working directory cleaned

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
    - Add `paperCommit = 5395ae37bd372235390d28292ed582d0c4fc23dd`
    - Keep existing mcVersion and version properties

6. **Create new module directories**
   ```
   mkdir -p aincrad-api/paper-patches/{files,features}
   mkdir -p aincrad-server/{paper-patches/{files,features},minecraft-patches/{sources,features}}
   mkdir -p aincrad-server/src/main/java
   ```

### Phase 3: Extract Custom Patches

**All 6 custom patches belong in `aincrad-server/paper-patches/features/` in this order:**

| # | Filename | Original |
|---|----------|----------|
| 0001 | Parallel-World-Ticking-SP | 0032 |
| 0002 | Network-Modifications | 0033 |
| 0003 | Water-World-Modifications | 0034 |
| 0004 | Remove-End-Dragon-Battle | 0035 |
| 0005 | Giants-AI | 0036 |
| 0006 | Per-world-Monster-Limits | 0037 |

THEN YOU FAILED MISERABLY, SO WE ARE RESTARTING.
