# AGENT INSTRUCTIONS

## Purpose
This repository is a multithreaded fork of the Paper high-performance Minecraft server implementation focused on stability and new custom content.

## High-Level Directives
- **Maintain an updated list of files you have read in your context**
- **After reading a file, IMMEDIATELY summarize your findings and next actions**
- **STAY FOCUSED! After each action you should ask yourself "how does this relate to the initial request?"**

## Key Files and Project Structure
- Original Minecraft code is located at paper-server/src/minecraft/java/net/minecraft
  - This code already has all the patches applied from the paper-server/patches dir.
  - Entrypoint of the Java application is the runServer() method in paper-server/src/minecraft/java/net/minecraft/server/MinecraftServer.java
  - Most main loop logic for the game gets called from either tickServer() or tickChildren() in paper-server/src/minecraft/java/net/minecraft/server/MinecraftServer.java
  - **Multi-threading**
    - The Vanilla chunk system has been replaced with a multi-threaded version, "ca.spottedleaf.moonrise".
    - World ticking has been parallelized by assigning each ServerLevel instance its own tickExecutor from org.evlis.ServerLevelTickExecutorThreadFactory, limited by a global serverLevelTickingSemaphore that is initialized in paper-server/src/minecraft/java/net/minecraft/server/dedicated/DedicatedServer.java
    - World creation/initialization has been moved out of MinecraftServer.java and into ServerLevel.java to allow creation and execution to happen off the main thread.
- The PaperAPI code is under paper-api/src/main/java
