# Plugin-Controllable Raids — Implementation Plan

## Goal

Expose two API operations to plugins:
1. **Create Raid** — start a raid at an arbitrary `BlockPos` (not necessarily a village), triggered by a player
2. **Stop Raid** — forcibly end a raid with an explicit victory or failure outcome

## Changes Required

### 1. `Raid.java` — Add `pluginControlled` flag

Add a `public boolean pluginControlled = false;` field.

In `Raid.tick()`, guard only the `isVillage(center)` loss/stop block with this flag:

```java
if (!this.pluginControlled && !this.level.isVillage(this.center)) {
    this.moveRaidCenterToNearbyVillageSection();
}
if (!this.pluginControlled && !this.level.isVillage(this.center)) {
    if (this.groupsSpawned > 0) { ... LOSS ... }
    else { ... STOP NOT_IN_VILLAGE ... }
}
```

The difficulty-to-peaceful check and 48,000-tick timeout are intentionally left unchanged.

Also add a `forceStop(boolean victory)` method:

```java
// Aincrad start
public void forceStop(boolean victory) {
    if (victory) {
        this.status = RaidStatus.VICTORY;
        for (UUID uuid : this.heroesOfTheVillage) {
            Entity entity = this.level.getEntity(uuid);
            if (entity instanceof LivingEntity living && !entity.isSpectator()) {
                living.addEffect(new MobEffectInstance(
                    MobEffects.HERO_OF_THE_VILLAGE, 48000, this.raidOmenLevel - 1, false, false, true));
                if (living instanceof ServerPlayer sp) {
                    sp.awardStat(Stats.RAID_WIN);
                    CriteriaTriggers.RAID_WIN.trigger(sp);
                }
            }
        }
        List<org.bukkit.entity.Player> winners = this.heroesOfTheVillage.stream()
            .map(this.level::getEntity)
            .filter(e -> e instanceof ServerPlayer)
            .map(e -> ((ServerPlayer) e).getBukkitEntity())
            .collect(java.util.stream.Collectors.toList());
        org.bukkit.craftbukkit.event.CraftEventFactory.callRaidFinishEvent(this, winners);
    } else {
        this.status = RaidStatus.LOSS;
        org.bukkit.craftbukkit.event.CraftEventFactory.callRaidFinishEvent(this, new java.util.ArrayList<>());
    }
}
// Aincrad end
```

### 2. `Raids.java` — Add `createRaid(ServerPlayer, BlockPos, int, boolean)` overload

Add a new overload alongside `createOrExtendRaid` that accepts a `pluginControlled` boolean and an
explicit bad omen level, bypassing `absorbRaidOmen` and the village centroid averaging:

```java
// Aincrad start
@Nullable
public Raid createRaid(ServerPlayer player, BlockPos pos, int badOmenLevel, boolean pluginControlled) {
    if (player.isSpectator()) return null;
    if (this.level.getGameRules().getBoolean(GameRules.RULE_DISABLE_RAIDS)) return null;
    if (!player.level().dimensionType().hasRaids()) return null;

    Raid raid = this.getOrCreateRaid(player.serverLevel(), pos);

    if (!org.bukkit.craftbukkit.event.CraftEventFactory.callRaidTriggerEvent(raid, player)) {
        return null;
    }

    if (!this.raidMap.containsKey(raid.getId())) {
        this.raidMap.put(raid.getId(), raid);
    }

    raid.setRaidOmenLevel(badOmenLevel);
    raid.pluginControlled = pluginControlled;
    this.setDirty();
    return raid;
}
// Aincrad end
```

Note: Village POI centroid averaging is intentionally skipped — the caller passes the exact center.

### 3. `CraftRaid.java` — Implement `forceStop(boolean)`

```java
// Aincrad start
@Override
public void forceStop(boolean victory) {
    this.handle.forceStop(victory);
}
// Aincrad end
```

### 4. `CraftWorld.java` — Implement `createRaid`

```java
// Aincrad start
@Override
public org.bukkit.Raid createRaid(org.bukkit.entity.Player player, org.bukkit.Location location,
                                   int badOmenLevel, boolean pluginControlled) {
    ServerPlayer serverPlayer = ((CraftPlayer) player).getHandle();
    BlockPos pos = CraftLocation.toBlockPosition(location);
    net.minecraft.world.entity.raid.Raid raid =
        getHandle().getRaids().createRaid(serverPlayer, pos, badOmenLevel, pluginControlled);
    return raid != null ? new CraftRaid(raid) : null;
}
// Aincrad end
```

### 5. `aincrad-api` — Extend `org.bukkit.Raid` and `org.bukkit.World`

**`org.bukkit.Raid` additions:**
```java
// Aincrad start
void forceStop(boolean victory);
boolean isPluginControlled();
// Aincrad end
```

**`org.bukkit.World` additions:**
```java
// Aincrad start
@Nullable
org.bukkit.Raid createRaid(@NotNull Player player, @NotNull Location location,
                            int badOmenLevel, boolean pluginControlled);
// Aincrad end
```

## What Is Intentionally Unchanged

- `RaidTriggerEvent` is still fired (and still cancellable) — a player is always required
- Peaceful difficulty and 48,000-tick timeout still terminate plugin-controlled raids
- Wave composition, spawn positions, and buff logic are untouched
- `createOrExtendRaid` is untouched — existing vanilla behaviour is preserved

## File Touch List

| File | Change |
|------|--------|
| `aincrad-server/.../raid/Raid.java` | Add `pluginControlled` field, guard `isVillage` checks, add `forceStop()` |
| `aincrad-server/.../raid/Raids.java` | Add `createRaid()` overload |
| `paper-server/.../CraftRaid.java` | Implement `forceStop()` |
| `paper-server/.../CraftWorld.java` | Implement `createRaid()` |
| `aincrad-api` patches | Add `forceStop()` + `isPluginControlled()` to `Raid`, add `createRaid()` to `World` |

