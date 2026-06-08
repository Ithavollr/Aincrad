# Custom Potion Brewing System

## Goal
Create a server-side potion brewing system that works with custom effects and durations without requiring client-side modifications or registry changes. This enables:
- Custom potions (e.g., Wither potion) to be brewable
- Per-item duration modifications via plugins
- Unified effect-based brewing ruleset

## Architecture

### Key Insight
The current potion registry system cannot be modified without breaking client compatibility. Splash/lingering potions are determined by item type, not registry entries. We can bypass the registry entirely by using custom effects and server-side flags.

### Solution
Replace registry-based brewing with an effect-based system:
- Store custom effects directly in `PotionContents.customEffects`
- Use server-side NBT flag to track if a potion has been upgraded
- Apply brewing transformations based on effect rules rather than registry lookups
- Keep item type conversions (PotionItem → SplashPotionItem → LingeringPotionItem) separate from effect transformations

### Brewing Rules
- **Redstone**: Duration × 2 (if upgradeable)
- **Glowstone**: Amplifier + 1 or Duration / 2 (depending on effect type)
- **Gunpowder**: Convert PotionItem → SplashPotionItem
- **Dragon Breath**: Convert SplashPotionItem → LingeringPotionItem
- **Awkward**: Base potion for custom effect additions

### Upgrade Guard
Each potion can only be upgraded once per upgrade path (duration or amplifier). The system must check:
1. If the potion has the NBT "upgraded" flag set
2. If the potion is an upgraded registry potion (e.g., LONG_SLOW_FALLING) - these should be treated as already upgraded even without the flag
3. Datapacks and commands can spawn registry potions, so the guard must recognize upgraded registry potion types and ignore them

## Implementation Plan

### 1. Brewing Stand Logic
Modify `BrewingStandBlockEntity.doBrew()` to:
- Check if the input potion is upgradeable (no upgraded flag, not a registry upgraded variant)
- Apply effect transformations based on ingredient
- Set the "upgraded" NBT flag after successful upgrade
- Handle item type conversions (splash/lingering) separately

### 2. Effect Transformation Table
Create a mapping system for ingredient → effect transformation:
```java
Map<Item, EffectTransformation> transformations = Map.of(
    Items.REDSTONE, new EffectTransformation(DurationMultiplier(2.0)),
    Items.GLOWSTONE_DUST, new EffectTransformation(AmplifierIncrement(1)),
    Items.GUNPOWDER, new ItemTypeTransformation(PotionItem → SplashPotionItem),
    Items.DRAGON_BREATH, new ItemTypeTransformation(SplashPotionItem → LingeringPotionItem)
);
```

### 3. NBT Storage
Store the "upgraded" flag in item NBT (outside `PotionContents` to avoid CODEC changes):
```java
itemStack.getOrCreateTag().putBoolean("AincradUpgraded", true);
```

### 4. API for Plugins
Add API methods to `PotionMeta`:
- `boolean isUpgraded()` - checks NBT flag and registry potion type
- `void setUpgraded(boolean)` - sets NBT flag
- Existing `getBaseDuration()` / `setBaseDuration()` for per-item duration control

### 5. Registry Potion Detection
Add logic to detect if a potion is a registry upgraded variant:
- Check if `PotionContents.potion` is set to a LONG_ or STRONG_ variant
- If so, treat as already upgraded for that path
- This handles datapacks/commands spawning registry potions

## Data Flow

```
Brewing starts:
  Input potion → check upgradeable (no flag, not registry upgraded)
  Ingredient → lookup transformation rule
  Apply transformation to custom effects
  Set "AincradUpgraded" NBT flag
  Output potion with modified effects

Plugin modifies via BrewEvent:
  PotionMeta.setBaseDuration(5400) → custom effects with new duration
  PotionMeta.setUpgraded(true) → mark as upgraded

Item is saved:
  Custom effects stored in PotionContents.customEffects
  "AincradUpgraded" flag stored in item NBT

Item is used:
  getAllEffects() → returns custom effects
  Tooltip → displays custom effects
  Application → applies custom effects

Brewing guard:
  Check "AincradUpgraded" flag OR registry potion type
  If already upgraded → reject ingredient
  If not upgraded → apply transformation
```

## Advantages
- No network protocol changes → client compatibility preserved
- No registry modifications → no client mod required
- Custom potions fully supported
- Plugins can create custom potions via API
- Unified ruleset for all brewing operations

## Caveats
- Must handle existing registry potions (LONG_SLOW_FALLING, etc.) as already upgraded
- Datapacks and commands can spawn registry potions, so the upgrade guard must check both NBT flag and registry potion type
- Existing plugins relying on registry potion types may need updates
