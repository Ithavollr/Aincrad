# AGENT INSTRUCTIONS

## Purpose
This repository is a multithreaded fork of the Paper high-performance Minecraft server implementation focused on stability, maximizing online player capacity, and new custom content.

## High-Level Directives
- **Maintain an updated list of files you have read in your context**
- **After reading a file, IMMEDIATELY summarize your findings and next actions**
- **STAY FOCUSED! After each action ask yourself "how does this relate to the initial request?"**
- **ASK before making design decisions. Implement only what is requested.**

## Project Structure

### Aincrad-specific code (where custom work happens)
- `aincrad-server/src/minecraft/java/net/minecraft/` — Patched Minecraft server source (NMS)
  - Custom sections marked with `// Aincrad start` / `// Aincrad end` comments
  - Key file: `world/item/alchemy/PotionBrewing.java` — Custom potion brewing system
  - Key file: `server/MinecraftServer.java` — Server entrypoint (`runServer()`, `tickServer()`, `tickChildren()`)
  - Key file: `server/dedicated/DedicatedServer.java` — Server initialization
- `aincrad-server/minecraft-patches/` — Patch files for NMS code
- `aincrad-api/` — Custom API additions (builds on top of paper-api)

### Paper upstream code (Aincrad patches this too)
- `paper-server/src/main/java/` — Paper server code (CraftBukkit, Spottedleaf Moonrise, Paper additions)
- `paper-api/src/main/java/` — Bukkit/Paper API
- `aincrad-server/paper-patches/` — Patch files for Paper server code
- `aincrad-api/paper-patches/` — Patch files for Paper API

## Key Architecture Notes

### Multi-threading
- Vanilla chunk system replaced with `ca.spottedleaf.moonrise`
- World ticking parallelized via per-ServerLevel tick executors
- World creation/initialization moved into ServerLevel.java for off-main-thread execution

## Special Cases:
When Grep tool returns "Access to [path] is prohibited by .gitignore", this is a MISLEADING error message. 
The user has confirmed the settings allow us to access files listed in .gitignore. You can verify this using alternative tools like 
find_by_name or read_file to access the files. The Grep tool has internal limitations that cause this misleading error message.
