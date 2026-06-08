# Custom Potion Brewing System

## Goal
Create a server-side potion brewing system where ALL potions with effects are custom-effect-based from the moment of brewing. Only WATER and AWKWARD remain as registry potions. This enables:
- Every brewed potion to be plugin-modifiable (plugins can only modify customEffects, not registry effects)
- Custom potions (e.g., Wither potion) to be brewable
- Per-item duration modifications via plugins
- Unified effect-based brewing ruleset

## Architecture

### Key Insight
Plugins can only modify `PotionContents.customEffects`, not registry potion effects. By making ALL effect potions custom-effect-based from the initial brew (e.g., Awkward + Spider Eye → custom-effect Poison with 900 ticks), every potion becomes plugin-modifiable. Only WATER and AWKWARD stay as registry potions since they have no effects to modify.

### Solution
Replace registry-based brewing with an effect-based system:
- **Base recipes** (Awkward + ingredient): produce custom-effect potions with `customName` for tooltip (no registry potion reference)
- Store custom effects directly in `PotionContents.customEffects`
- Use server-side NBT flag to track if a potion has been upgraded
- Apply brewing transformations based on effect rules rather than registry lookups
- Keep item type conversions (PotionItem → SplashPotionItem → LingeringPotionItem) separate from effect transformations

### Brewing Rules
- **Redstone**: Duration × 2 for all effects (if potion is upgradeable)
- **Glowstone**: Amplifier + 1 for all effects (if potion is upgradeable)
- **Gunpowder**: Convert PotionItem → SplashPotionItem
- **Dragon Breath**: Convert SplashPotionItem → LingeringPotionItem
- **Awkward + ingredient**: Produces custom-effect potion (e.g., Awkward + Spider Eye → custom Poison, 900 ticks)

### Upgrade Guard
Each potion can only be upgraded once per upgrade path (duration or amplifier). The system must check:
1. If the potion has the NBT "upgraded" flag set
2. If the potion is an upgraded registry potion (e.g., LONG_SLOW_FALLING) - these should be treated as already upgraded even without the flag
    - LONG_ and STRONG_ variants are upgraded potions, all others must be treated as upgradeable.
3. Datapacks and commands can spawn registry potions, so the guard must recognize upgraded registry potion types and ignore them

## Implementation Plan
NOTE: for all new code blocks, encapsulate in `// Aincrad start` and `// Aincrad end` like the existing Paper changes. For single line changes, use `// Aincrad <description>`.

### 1. Base Recipe Table
Add a static map in `PotionBrewing` mapping ingredient → base potion definition (effect, duration, name).
This replaces all `addStartMix()` and most `addMix()` calls. Only keep `WATER + NETHER_WART → AWKWARD`.
Container recipes (`addContainerRecipe` for Gunpowder/Dragon Breath) remain unchanged.

### 2. Modify `mix()` — Base Brewing + Upgrades
In `PotionBrewing.mix()`, add logic before the existing registry lookups:
- If input is AWKWARD and ingredient is in the base recipe table: produce a custom-effect potion
- If ingredient is Redstone/Glowstone and input has custom effects and is not upgraded: apply duration×2 or amplifier+1, set upgraded flag

### 3. Modify `hasMix()` and `isIngredient()`
Accept base recipe ingredients for AWKWARD potions, and Redstone/Glowstone for upgradeable custom-effect potions.

### 3. NBT Storage
Store the "upgraded" flag in item NBT (outside `PotionContents` to avoid CODEC changes):
```java
itemStack.getOrCreateTag().putBoolean("PotionHasBeenUpgraded", true);
```

### 4. API for Plugins
Document process for plugin creators:
- what exact NBT flag is used for upgrades?
- what effects lists correspond to which potions in the new system.
- NO API changes for this! Plugins will use the existing customeffects system, as all new potions will now be custom effect-based.

### 5. Registry Potion Detection
Add logic to detect if a potion is a registry upgraded variant:
- Check if `PotionContents.potion` is set to a LONG_ or STRONG_ variant
- If so, treat as already upgraded for that path
- This handles datapacks/commands spawning registry potions

## Data Flow

```
Base brewing (Awkward + Spider Eye):
  Input: Awkward potion (registry)
  Ingredient: Spider Eye → lookup base recipe → "poison", POISON, 900 ticks
  Output: PotionContents(potion=empty, customEffects=[POISON/900], customName="poison")

Upgrade brewing (Poison + Redstone):
  Input: custom-effect Poison potion
  Check: PotionHasBeenUpgraded → false, proceed
  Apply: duration × 2 → POISON/1800
  Set: PotionHasBeenUpgraded = true
  Output: upgraded custom-effect Poison potion

Plugin modifies via BrewEvent:
  PotionMeta.setBaseDuration(5400) → custom effects with new duration
  PotionMeta.setUpgraded(true) → mark as upgraded

Item is saved:
  Custom effects stored in PotionContents.customEffects
  "PotionHasBeenUpgraded" flag stored in item NBT

Item is used:
  getAllEffects() → returns custom effects
  Tooltip → displays custom effects
  Application → applies custom effects

Brewing guard:
  Check "PotionHasBeenUpgraded" flag OR registry potion type
  If already upgraded → reject ingredient
  If not upgraded → apply transformation
```

## Advantages
- No network protocol changes → client compatibility preserved
- No registry modifications → no client mod required
- Custom potions fully supported
- Plugins can create custom potions via API, no API changes required.
- Unified ruleset for all brewing operations

## Caveats
- Must handle existing registry potions (LONG_SLOW_FALLING, etc.) as already upgraded
- Datapacks and commands can spawn registry potions, so the upgrade guard must check both NBT flag and registry potion type
