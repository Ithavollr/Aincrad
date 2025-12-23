# AGENT INSTRUCTIONS

## Purpose
This repository is a multithreaded fork of the Paper high-performance Minecraft server implementation.
Changes that break vanilla behavior, plugin compatibility, or upstream mergeability are unacceptable unless explicitly requested.

## High-Level Constraints
- **Do not change behavior unless explicitly instructed**
- **Do not introduce new dependencies without approval**
If requirements are ambiguous, stop and ask.

## Memory Bank Protocol
I am an agent with a limited context window. To maintain continuity:

1. **Start of Task** I MUST read `docs/memory.md` to understand the current state.
2. **End of Task** Before finishing, I MUST update, or ask the user to update `memory.md` with:
    - What was accomplished.
    - What is left to do.
    - Any architectural decisions made.

## Summary Directive

Be conservative.
Be explicit.
Minimize diff.
Preserve intent.
