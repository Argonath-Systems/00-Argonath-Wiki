# Combat Framework Migration Guide

> **Audience**: Developers migrating from Combat Framework v1.x to v2.0+  
> **Migration Type**: BREAKING CHANGES (Lombok → Records, Stats Integration)  
> **Estimated Time**: 1-2 hours for typical mod  
> **Last Updated**: 2026-01-27

---

## Table of Contents

1. [Overview](#overview)
2. [Breaking Changes Summary](#breaking-changes-summary)
3. [Step-by-Step Migration](#step-by-step-migration)
4. [Code Migration Patterns](#code-migration-patterns)
5. [Testing Checklist](#testing-checklist)
6. [Troubleshooting](#troubleshooting)

---

## Overview

### What Changed?

Combat Framework **v2.0.0** introduces two major architectural changes:

1. **Lombok → Java 25 Records**
   - All data classes converted from Lombok `@Data`/`@Builder` to records
   - Affects getter syntax, constructors, and builder patterns
   - Reason: Java 25 compatibility (Lombok 1.18.36 has fatal errors with TypeTag::UNKNOWN)

2. **Stats Framework Integration**
   - Removed duplicate stat calculation system
   - Now uses `04-framework-stats` with `StatContainer` and `Modifier` system
   - Deleted: `StatCalculator`, `CombatModifier`, `BaseStat`, `CalculatedStat`
   - Added: `CombatStats` (40+ `StatId` constants)

### Why Migrate?

- ✅ **Java 25 Compatibility**: No more Lombok fatal errors
- ✅ **Cleaner Architecture**: Single source of truth for stats
- ✅ **Better Performance**: Records are optimized by JVM
- ✅ **Type Safety**: Compiler-enforced immutability
- ✅ **Modern Code**: Records are idiomatic Java 16+

### Migration Complexity

| Mod Complexity | Estimated Time | Notes |
|----------------|----------------|-------|
| **Simple** (Uses CombatService only) | 30 min | Mostly syntax changes |
| **Medium** (Custom abilities/effects) | 1-2 hours | Need to update stat references |
| **Complex** (Custom stat calculations) | 2-4 hours | May need refactoring |

---

## Breaking Changes Summary

### 1. Getter Syntax Changes

**All record accessors drop the `get` prefix and `is` prefix for booleans.**

| Old (Lombok) | New (Record) | Notes |
|--------------|--------------|-------|
| `event.getRawDamage()` | `event.rawDamage()` | Drop `get` |
| `event.getEventId()` | `event.eventId()` | Drop `get` |
| `event.isCanDodge()` | `event.canDodge()` | Drop `is` |
| `event.isAttack()` | `event.isAttack()` | Keep `is` for `isX()` field name |
| `result.getTotalDamage()` | `result.totalDamage()` | Drop `get` |
| `result.isWasCritical()` | `result.wasCritical()` | Drop `is` |

### 2. Builder Pattern Removed

**Records don't have builders. Use constructors or factory methods.**

| Old (Lombok) | New (Record) |
|--------------|--------------|
| `DamageEvent.builder().attacker(a).target(t).build()` | `DamageEvent.create(a, t, damageMap)` |
| `AbilityResult.builder().success(true).build()` | `new AbilityResult(id, true, null, ...)` |

### 3. Stat System Changes

**Complete API replacement.**

| Old API | New API | Notes |
|---------|---------|-------|
| `StatCalculator.calculate(entity)` | `CombatStats.calculateDerivedStats(stats)` | Stats Framework |
| `entity.getStrength()` | `entity.getStats().get(CombatStats.STRENGTH)` | Via StatContainer |
| `new CombatModifier(stat, value)` | `new FlatModifier(id, stat, name, source, value)` | Stats Framework |
| `entity.addModifier(modifier)` | `entity.getStats().addModifier(modifier)` | Via StatContainer |
| `entity.getCriticalChance()` | `entity.getStats().get(CombatStats.CRITICAL_CHANCE)` | Derived stat |

### 4. Deleted Classes

**These classes no longer exist:**

- `com.argonathsystems.combat.stat.StatCalculator` → Use `CombatStats.calculateDerivedStats()`
- `com.argonathsystems.combat.stat.CombatModifier` → Use `FlatModifier`/`PercentModifier`
- `com.argonathsystems.combat.stat.BaseStat` → Use `StatContainer.set(statId, value)`
- `com.argonathsystems.combat.stat.CalculatedStat` → Use `StatContainer.get(statId)`

### 5. New Dependencies

**Add to your POM:**

```xml
<dependency>
    <groupId>com.argonathsystems.framework</groupId>
    <artifactId>argonath-stats</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

---

## Step-by-Step Migration

### Step 1: Update Dependencies

Edit your `pom.xml`:

```xml
<dependencies>
    <!-- Update combat framework version -->
    <dependency>
        <groupId>com.argonathsystems.framework</groupId>
        <artifactId>argonath-helm-combat</artifactId>
        <version>2.0.0-SNAPSHOT</version> <!-- Was 1.x -->
    </dependency>
    
    <!-- NEW: Add stats framework -->
    <dependency>
        <groupId>com.argonathsystems.framework</groupId>
        <artifactId>argonath-stats</artifactId>
        <version>1.0.0-SNAPSHOT</version>
    </dependency>
</dependencies>
```

### Step 2: Update Imports

Replace old imports:

```java
// ❌ DELETE these imports
import com.argonathsystems.combat.stat.StatCalculator;
import com.argonathsystems.combat.stat.CombatModifier;
import com.argonathsystems.combat.stat.BaseStat;
import com.argonathsystems.combat.stat.CalculatedStat;

// ✅ ADD these imports
import com.argonathsystems.combat.domain.stats.CombatStats;
import com.argonathsystems.framework.stats.StatContainer;
import com.argonathsystems.framework.stats.FlatModifier;
import com.argonathsystems.framework.stats.PercentModifier;
import com.argonathsystems.framework.stats.StatId;
```

### Step 3: Update Getter Calls

Run find-replace across your codebase:

| Find | Replace | Regex |
|------|---------|-------|
| `.getRawDamage()` | `.rawDamage()` | No |
| `.getEventId()` | `.eventId()` | No |
| `.getAbilityId()` | `.abilityId()` | No |
| `.getTotalDamage()` | `.totalDamage()` | No |
| `.isCanDodge()` | `.canDodge()` | No |
| `.isCanCrit()` | `.canCrit()` | No |
| `.isWasCritical()` | `.wasCritical()` | No |
| `.isWasDodged()` | `.wasDodged()` | No |

**Regex Alternative** (for bulk replacement):
```regex
Find: \.get([A-Z][a-zA-Z]+)\(\)
Replace: .$1()  # Then lowercase first letter manually
```

### Step 4: Replace Builder Calls

#### DamageEvent

```java
// ❌ OLD
DamageEvent event = DamageEvent.builder()
    .eventId(UUID.randomUUID())
    .attacker(attacker)
    .target(target)
    .rawDamage(Map.of(DamageType.PHYSICAL, 100.0))
    .isAttack(true)
    .canCrit(true)
    .build();

// ✅ NEW (Factory Method)
DamageEvent event = DamageEvent.create(
    attacker,
    target,
    Map.of(DamageType.PHYSICAL, 100.0)
);

// ✅ OR (Full Constructor)
DamageEvent event = new DamageEvent(
    UUID.randomUUID(),
    attacker,
    target,
    Map.of(DamageType.PHYSICAL, 100.0),
    true,   // isAttack
    false,  // isSpell
    false,  // isProjectile
    false,  // isArea
    true,   // canCrit
    true,   // canDodge
    true,   // canBlock
    null,   // abilityId
    Instant.now()
);
```

#### AbilityResult

```java
// ❌ OLD
AbilityResult result = AbilityResult.builder()
    .executionId(UUID.randomUUID())
    .success(true)
    .damageEvents(events)
    .build();

// ✅ NEW
AbilityResult result = new AbilityResult(
    UUID.randomUUID(),
    true,               // success
    null,               // failureReason
    1,                  // castCount
    events,             // damageEvents
    List.of(),          // healEvents
    List.of(),          // effectsApplied
    Instant.now()
);
```

### Step 5: Migrate Stat System

#### Before (Old API)

```java
public class OldCombatEntity {
    private double strength = 10.0;
    private double criticalChance;
    
    public void initialize() {
        // Calculate stats
        StatCalculator calculator = new StatCalculator();
        CalculatedStat critStat = calculator.calculateCriticalChance(this);
        this.criticalChance = critStat.getValue();
    }
    
    public void addEquipment(Equipment equipment) {
        CombatModifier modifier = new CombatModifier("equipment", 50.0);
        this.strength += modifier.getValue();
    }
    
    public double getStrength() {
        return strength;
    }
}
```

#### After (New API)

```java
public class NewCombatEntity implements CombatEntity {
    private final StatContainer stats = new StatContainer();
    
    public void initialize() {
        // Initialize base stats
        CombatStats.initializeDefaults(stats);
        
        // Calculate derived stats
        CombatStats.calculateDerivedStats(stats);
    }
    
    public void addEquipment(Equipment equipment) {
        FlatModifier modifier = new FlatModifier(
            UUID.randomUUID(),
            CombatStats.STRENGTH,
            "Legendary Sword",
            "equipment:main_hand",
            50.0
        );
        stats.addModifier(modifier);
    }
    
    @Override
    public StatContainer getStats() {
        return stats;
    }
    
    // Stats accessed via:
    // double str = entity.getStats().get(CombatStats.STRENGTH);
}
```

### Step 6: Update Effect Application

#### Before

```java
// Old custom buff system
public void applyStrengthBuff(Entity entity, double amount, Duration duration) {
    StrengthBuff buff = new StrengthBuff(amount, duration);
    entity.addBuff(buff);
}
```

#### After

```java
// New ActiveEffect + Modifier system
public void applyStrengthBuff(CombatEntity entity, double amount, Duration duration) {
    List<Modifier> modifiers = List.of(
        new FlatModifier(
            UUID.randomUUID(),
            CombatStats.STRENGTH,
            "Strength Buff",
            "buff:strength",
            amount
        )
    );
    
    ActiveEffect buff = new ActiveEffect(
        "buff_" + UUID.randomUUID(),
        "buff_system",
        EffectCategory.BUFF,
        modifiers,
        Instant.now(),
        Instant.now().plus(duration),
        1,      // stacks
        1,      // maxStacks
        null,   // no periodic effect
        "strength_icon.png",
        true    // show in UI
    );
    
    effectService.applyEffect(entity, buff);
}
```

### Step 7: Update CombatEntity Implementation

Implement the new `CombatEntity` interface:

```java
public class HytaleCombatEntity implements CombatEntity {
    private final Entity hytaleEntity;
    private final StatContainer stats;
    private final List<ActiveEffect> effects;
    
    public HytaleCombatEntity(Entity hytaleEntity) {
        this.hytaleEntity = hytaleEntity;
        this.stats = new StatContainer();
        this.effects = new ArrayList<>();
        
        // Initialize
        CombatStats.initializeDefaults(stats);
        CombatStats.calculateDerivedStats(stats);
    }
    
    @Override
    public StatContainer getStats() {
        return stats;
    }
    
    @Override
    public List<ActiveEffect> getActiveEffects() {
        return Collections.unmodifiableList(effects);
    }
    
    @Override
    public void addEffect(ActiveEffect effect) {
        effects.add(effect);
    }
    
    @Override
    public void removeEffect(ActiveEffect effect) {
        effects.remove(effect);
    }
    
    @Override
    public EquipmentSnapshot getEquipment() {
        // Map Hytale equipment to snapshot
        return buildEquipmentSnapshot();
    }
    
    @Override
    public void recalculateStats() {
        CombatStats.calculateDerivedStats(stats);
    }
}
```

### Step 8: Clean Build

```bash
cd /mnt/d/Gaming/Argonath-Systems
just clean
just build-mods
```

---

## Code Migration Patterns

### Pattern 1: Damage Calculation

#### Before

```java
double damage = event.getRawDamage().get(DamageType.PHYSICAL);
double strength = attacker.getStrength();
double critChance = attacker.getCriticalChance();

if (Math.random() < critChance) {
    damage *= 1.5;
}
```

#### After

```java
double damage = event.rawDamage().get(DamageType.PHYSICAL);
StatContainer stats = attacker.getStats();
double strength = stats.get(CombatStats.STRENGTH);
double critChance = stats.get(CombatStats.CRITICAL_CHANCE);

if (Math.random() < critChance) {
    double critDamage = stats.get(CombatStats.CRITICAL_DAMAGE);
    damage *= critDamage;
}
```

### Pattern 2: Ability Execution

#### Before

```java
AbilityResult result = AbilityResult.builder()
    .success(true)
    .damageEvents(List.of(damageEvent))
    .build();

return CompletableFuture.completedFuture(result);
```

#### After

```java
AbilityResult result = new AbilityResult(
    UUID.randomUUID(),
    true,               // success
    null,               // failureReason
    1,                  // castCount
    List.of(damageEvent), // damageEvents
    List.of(),          // healEvents
    List.of(),          // effectsApplied
    Instant.now()
);

return CompletableFuture.completedFuture(result);
```

### Pattern 3: Equipment Stats

#### Before

```java
public void onEquipItem(Player player, Item item) {
    double bonusStrength = item.getStatBonus("strength");
    player.setStrength(player.getStrength() + bonusStrength);
}

public void onUnequipItem(Player player, Item item) {
    double bonusStrength = item.getStatBonus("strength");
    player.setStrength(player.getStrength() - bonusStrength);
}
```

#### After

```java
public void onEquipItem(Player player, Item item) {
    CombatEntity entity = getCombatEntity(player);
    StatContainer stats = entity.getStats();
    
    // Add modifiers
    for (StatBonus bonus : item.getStatBonuses()) {
        Modifier modifier = new FlatModifier(
            UUID.randomUUID(),
            bonus.statId(),
            item.getName(),
            "equipment:" + item.getSlot(),
            bonus.value()
        );
        stats.addModifier(modifier);
    }
    
    entity.recalculateStats();
}

public void onUnequipItem(Player player, Item item) {
    CombatEntity entity = getCombatEntity(player);
    StatContainer stats = entity.getStats();
    
    // Remove all modifiers from this slot
    stats.removeModifiersBySource("equipment:" + item.getSlot());
    entity.recalculateStats();
}
```

### Pattern 4: Periodic Effects

#### Before

```java
public void tickDotEffects(Entity entity) {
    for (DotEffect dot : entity.getDoTEffects()) {
        if (dot.shouldTick()) {
            dealDamage(entity, dot.getDamagePerTick());
            dot.recordTick();
        }
    }
}
```

#### After

```java
public void tickPeriodicEffects(CombatEntity entity) {
    // Use EffectService
    effectService.tickPeriodicEffects(entity);
    
    // Or manually:
    for (ActiveEffect effect : entity.getActiveEffects()) {
        PeriodicEffect periodic = effect.periodicEffect();
        if (periodic != null && Instant.now().isAfter(periodic.nextTickAt())) {
            DamageEvent event = DamageEvent.create(
                entity,  // self-damage
                entity,
                Map.of(periodic.damageType(), periodic.damagePerTick())
            );
            combatService.dealDamage(event);
            
            // Update next tick time
            // (EffectService handles this automatically)
        }
    }
}
```

---

## Testing Checklist

After migration, verify:

- [ ] **Compilation**: `just build-mods` succeeds
- [ ] **Unit Tests**: All tests pass
- [ ] **Damage Calculation**: Attacks deal correct damage
- [ ] **Critical Hits**: Crits calculate correctly
- [ ] **Dodge/Block**: Mitigation works
- [ ] **Equipment**: Stats update when equipping/unequipping
- [ ] **Buffs/Debuffs**: Effects apply and expire correctly
- [ ] **Abilities**: Custom abilities execute
- [ ] **Periodic Effects**: DoT/HoT ticks work
- [ ] **Stat Recalculation**: Derived stats update after modifier changes

### Manual Testing Script

```java
@Test
public void testMigration() {
    // 1. Create entity
    CombatEntity entity = new TestCombatEntity();
    CombatStats.initializeDefaults(entity.getStats());
    
    // 2. Check base stats
    assertEquals(10.0, entity.getStats().get(CombatStats.STRENGTH));
    
    // 3. Add modifier
    entity.getStats().addModifier(new FlatModifier(
        UUID.randomUUID(),
        CombatStats.STRENGTH,
        "Test Buff",
        "test",
        50.0
    ));
    
    assertEquals(60.0, entity.getStats().get(CombatStats.STRENGTH));
    
    // 4. Recalculate derived stats
    entity.recalculateStats();
    double expectedPhysPower = 60.0 * 2.0;  // STR * 2
    assertEquals(expectedPhysPower, entity.getStats().get(CombatStats.PHYSICAL_POWER));
    
    // 5. Create damage event (record syntax)
    DamageEvent event = DamageEvent.create(entity, entity, Map.of(DamageType.PHYSICAL, 100.0));
    assertNotNull(event.eventId());
    assertEquals(100.0, event.rawDamage().get(DamageType.PHYSICAL));
    
    // 6. Deal damage
    DamageResult result = combatService.dealDamage(event);
    assertTrue(result.totalDamage() > 0);
}
```

---

## Troubleshooting

### Error: "Cannot resolve method getRawDamage()"

**Cause**: Using old Lombok getter syntax.

**Fix**: Change to record accessor:
```java
// ❌ event.getRawDamage()
// ✅ event.rawDamage()
```

### Error: "Cannot resolve method builder()"

**Cause**: Records don't have builders.

**Fix**: Use constructor or factory method:
```java
// ❌ DamageEvent.builder().build()
// ✅ DamageEvent.create(attacker, target, damageMap)
```

### Error: "Cannot find symbol: StatCalculator"

**Cause**: Class deleted in v2.0.

**Fix**: Use `CombatStats` utility:
```java
// ❌ StatCalculator.calculate(entity)
// ✅ CombatStats.calculateDerivedStats(entity.getStats())
```

### Error: "Type mismatch: cannot convert from double to StatId"

**Cause**: Trying to set raw value instead of StatId.

**Fix**:
```java
// ❌ stats.set("strength", 10.0)
// ✅ stats.set(CombatStats.STRENGTH, 10.0)
```

### Error: Stats not updating after equipment change

**Cause**: Forgot to recalculate derived stats.

**Fix**: Call `recalculateStats()`:
```java
stats.addModifier(modifier);
entity.recalculateStats();  // ← Important!
```

### Warning: "Deprecated constructor"

**Cause**: Using old constructor signature.

**Fix**: Check new record signature in API docs:
- See [[api/framework-combat#domain-models]]
- Or use factory methods like `DamageEvent.create()`

---

## Support

- **Documentation**: [[guides/combat-framework-usage]]
- **API Reference**: [[api/framework-combat]]
- **Examples**: `00-Argonath-Samples/combat-examples/`
- **Discord**: #combat-framework channel
- **Issues**: Report on GitHub with `migration` tag

---

**Last Updated**: 2026-01-27  
**Migration Version**: v1.x → v2.0.0  
**Status**: Complete
