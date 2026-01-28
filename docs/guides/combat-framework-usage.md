# Combat Framework Usage Guide

> **Audience**: Developers integrating combat functionality into mods  
> **Framework**: `04-framework-combat` v2.0.0+  
> **Prerequisites**: Java 25, Maven, Guice dependency injection  
> **Last Updated**: 2026-01-27

---

## Table of Contents

1. [Introduction](#introduction)
2. [Quick Start](#quick-start)
3. [Core Concepts](#core-concepts)
4. [Common Patterns](#common-patterns)
5. [Advanced Usage](#advanced-usage)
6. [Troubleshooting](#troubleshooting)

---

## Introduction

The Combat Framework provides a complete combat system with:
- ✅ **Stats integration** - Uses Stats Framework (`04-framework-stats`)
- ✅ **Platform-agnostic** - Zero Hytale imports in framework layer
- ✅ **Modern Java** - Java 25 records (no Lombok)
- ✅ **Service-oriented** - Guice dependency injection
- ✅ **Extensible** - Ability system, effects, gem socketing

### Architecture Overview

```
Your Mod (06-mod-*)
    ↓ depends on
04-framework-combat (Combat logic)
    ↓ depends on
04-framework-stats (Stat management)
```

**Key Principle**: The framework has **ZERO Hytale imports**. All game engine integration happens in YOUR mod layer.

---

## Quick Start

### 1. Add Dependency

Add to your mod's `pom.xml`:

```xml
<dependency>
    <groupId>com.argonathsystems.framework</groupId>
    <artifactId>argonath-helm-combat</artifactId>
    <version>1.0.0-SNAPSHOT</version>
</dependency>
```

### 2. Configure Guice Module

```java
package com.example.mymod;

import com.argonathsystems.mod.combat.CombatModule;
import com.google.inject.AbstractModule;

public class MyModModule extends AbstractModule {
    @Override
    protected void configure() {
        // Install combat module
        install(new CombatModule());
        
        // Your mod bindings...
    }
}
```

### 3. Inject Services

```java
import com.argonathsystems.combat.service.CombatService;
import com.argonathsystems.combat.service.EffectService;
import javax.inject.Inject;

public class MyModPlugin {
    private final CombatService combatService;
    private final EffectService effectService;
    
    @Inject
    public MyModPlugin(CombatService combatService, 
                       EffectService effectService) {
        this.combatService = combatService;
        this.effectService = effectService;
    }
}
```

### 4. Create Combat Entity

Implement `CombatEntity` in your entity adapter:

```java
import com.argonathsystems.combat.domain.CombatEntity;
import com.argonathsystems.framework.stats.StatContainer;
import hytale.game.entity.Entity; // Hytale import OK in adapter

public class HytaleCombatEntity implements CombatEntity {
    private final Entity hytaleEntity;
    private final StatContainer stats;
    private final List<ActiveEffect> effects;
    
    public HytaleCombatEntity(Entity hytaleEntity) {
        this.hytaleEntity = hytaleEntity;
        this.stats = new StatContainer();
        this.effects = new ArrayList<>();
        
        // Initialize combat stats
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
        // Map Hytale equipment to EquipmentSnapshot
        return buildEquipmentSnapshot();
    }
}
```

### 5. Deal Damage

```java
// Create damage event
DamageEvent event = DamageEvent.create(
    attacker,  // CombatEntity
    target,    // CombatEntity
    Map.of(DamageType.PHYSICAL, 50.0)
);

// Calculate and apply damage
DamageResult result = combatService.dealDamage(event);

// Log result
logger.info("Dealt {} damage (crit: {})", 
    result.totalDamage(), 
    result.wasCritical()
);
```

---

## Core Concepts

### Combat Stats

Combat uses **40+ stats** from the Stats Framework. All stats are accessed via `StatContainer`:

```java
StatContainer stats = entity.getStats();

// Primary attributes
double str = stats.get(CombatStats.STRENGTH);
double agi = stats.get(CombatStats.AGILITY);
double intel = stats.get(CombatStats.INTELLIGENCE);

// Derived combat stats
double physPower = stats.get(CombatStats.PHYSICAL_POWER);
double critChance = stats.get(CombatStats.CRITICAL_CHANCE);
double attackSpeed = stats.get(CombatStats.ATTACK_SPEED);

// Vitals
double currentHp = stats.get(CombatStats.CURRENT_HEALTH);
double maxHp = stats.get(CombatStats.MAX_HEALTH);
```

### Modifiers

Equipment, buffs, and effects apply **modifiers** to stats:

```java
import com.argonathsystems.framework.stats.FlatModifier;
import com.argonathsystems.framework.stats.PercentModifier;

// Sword: +20 Strength
stats.addModifier(new FlatModifier(
    UUID.randomUUID(),
    CombatStats.STRENGTH,
    "Legendary Sword",
    "equipment:main_hand",
    20.0
));

// Buff: +10% Attack Speed
stats.addModifier(new PercentModifier(
    UUID.randomUUID(),
    CombatStats.ATTACK_SPEED,
    "Battle Frenzy",
    "buff:frenzy",
    0.10  // 10%
));

// Stats auto-recalculate on .get()
double newStr = stats.get(CombatStats.STRENGTH); // 10 + 20 = 30
```

### Java 25 Records

All data classes use **records** (not Lombok):

```java
// WRONG (Lombok syntax)
double damage = event.getRawDamage();      // ❌ No such method
boolean canDodge = event.isCanDodge();     // ❌ No such method

// CORRECT (Record syntax)
double damage = event.rawDamage();         // ✅ Record accessor
boolean canDodge = event.canDodge();       // ✅ Boolean drops "is"
UUID id = event.eventId();                 // ✅ Field name
```

### Effects

Effects apply temporary stat modifiers:

```java
import com.argonathsystems.combat.domain.effect.*;

// Create buff effect
List<Modifier> modifiers = List.of(
    new FlatModifier(
        UUID.randomUUID(),
        CombatStats.PHYSICAL_POWER,
        "Battle Cry",
        "buff:battle_cry",
        50.0
    )
);

ActiveEffect buff = new ActiveEffect(
    "battle_cry_" + UUID.randomUUID(),
    "player:warrior_tank",
    EffectCategory.BUFF,
    modifiers,
    Instant.now(),
    Instant.now().plusSeconds(10),  // Expires in 10s
    1,  // stacks
    1,  // maxStacks
    null,  // no periodic effect
    "battle_cry_icon.png",
    true  // show in UI
);

// Apply effect (auto-integrates with stats)
effectService.applyEffect(entity, buff);

// Effect removed automatically when expired,
// or manually:
effectService.removeEffect(entity, buff.effectId());
```

---

## Common Patterns

### Pattern 1: Initialize New Player

```java
public void onPlayerJoin(Player player) {
    // Create combat wrapper
    HytaleCombatEntity combatEntity = new HytaleCombatEntity(player);
    
    // Set race bonuses
    StatContainer stats = combatEntity.getStats();
    stats.addModifier(new FlatModifier(
        UUID.randomUUID(),
        CombatStats.STRENGTH,
        "Dwarf Racial",
        "race:dwarf",
        15.0  // Dwarves get +15 STR
    ));
    
    // Recalculate derived stats
    combatEntity.recalculateStats();
    
    // Store entity mapping
    playerCombatMap.put(player.getId(), combatEntity);
}
```

### Pattern 2: Apply Equipment

```java
public void onEquipItem(Player player, ItemStack item) {
    CombatEntity combatEntity = getCombatEntity(player);
    StatContainer stats = combatEntity.getStats();
    
    // Remove old equipment modifiers from slot
    stats.removeModifiersBySource("equipment:" + item.getSlot());
    
    // Add new equipment modifiers
    for (StatBonus bonus : item.getStatBonuses()) {
        Modifier modifier = bonus.isPercent() 
            ? new PercentModifier(
                UUID.randomUUID(),
                bonus.statId(),
                item.getName(),
                "equipment:" + item.getSlot(),
                bonus.value()
              )
            : new FlatModifier(
                UUID.randomUUID(),
                bonus.statId(),
                item.getName(),
                "equipment:" + item.getSlot(),
                bonus.value()
              );
        stats.addModifier(modifier);
    }
    
    // Recalculate
    combatEntity.recalculateStats();
}
```

### Pattern 3: Periodic Tick (Effects, Cooldowns)

```java
@EventListener
public void onServerTick(ServerTickEvent event) {
    // Tick all active combat entities
    for (CombatEntity entity : activeCombatEntities) {
        // Process periodic effects (DoT/HoT)
        effectService.tickPeriodicEffects(entity);
        
        // Clean up expired effects
        effectService.cleanupExpiredEffects(entity);
    }
}
```

### Pattern 4: Execute Ability

```java
public void onPlayerUseAbility(Player player, String abilityId, Entity target) {
    CombatEntity caster = getCombatEntity(player);
    CombatEntity targetEntity = getCombatEntity(target);
    
    // Execute ability (async)
    CompletableFuture<AbilityResult> future = combatService.executeAbility(
        caster,
        abilityId,
        AbilityTarget.single(targetEntity)
    );
    
    // Handle result
    future.thenAccept(result -> {
        if (!result.success()) {
            player.sendMessage("Failed: " + result.failureReason());
            return;
        }
        
        // Play VFX, sounds, etc.
        playAbilityEffects(player, abilityId);
        
        // Update UI
        updateAbilityCooldown(player, abilityId);
    });
}
```

### Pattern 5: Damage with Effects

```java
public DamageResult dealWeaponAttack(CombatEntity attacker, CombatEntity target) {
    StatContainer attackerStats = attacker.getStats();
    
    // Get weapon damage
    double weaponDamage = attackerStats.get(CombatStats.PHYSICAL_POWER);
    
    // Add random variance (90%-110%)
    double variance = 0.9 + Math.random() * 0.2;
    double finalDamage = weaponDamage * variance;
    
    // Create damage event
    DamageEvent event = new DamageEvent(
        UUID.randomUUID(),
        attacker,
        target,
        Map.of(DamageType.PHYSICAL, finalDamage),
        true,   // isAttack
        false,  // isSpell
        false,  // isProjectile
        false,  // isArea
        true,   // canCrit
        true,   // canDodge
        true,   // canBlock
        null,   // no ability
        Instant.now()
    );
    
    // Calculate and apply
    DamageResult result = combatService.dealDamage(event);
    
    // Apply on-hit effects if hit
    if (!result.wasDodged() && !result.wasBlocked()) {
        applyOnHitEffects(attacker, target, result);
    }
    
    return result;
}

private void applyOnHitEffects(CombatEntity attacker, CombatEntity target, 
                                DamageResult result) {
    // Example: 20% chance to apply bleed on crit
    if (result.wasCritical() && Math.random() < 0.20) {
        ActiveEffect bleed = createBleedEffect(attacker, 5.0, Duration.ofSeconds(8));
        effectService.applyEffect(target, bleed);
    }
}
```

---

## Advanced Usage

### Custom Ability Implementation

Create your own ability:

```java
import com.argonathsystems.combat.domain.ability.*;

public class FireballAbility implements Ability {
    
    @Override
    public AbilityDefinition getDefinition() {
        return new AbilityDefinition(
            "fireball",
            "Fireball",
            AbilityType.SPELL,
            List.of(
                new AbilityLevel(
                    1,
                    new ResourceCost(ResourceType.MANA, 30.0),
                    Duration.ofSeconds(5),  // cooldown
                    Duration.ofMillis(1500), // cast time
                    20.0,  // range
                    List.of(
                        new AbilityEffect(
                            EffectType.DAMAGE,
                            EffectTarget.SINGLE_ENEMY,
                            new DamageDefinition(
                                DamageType.FIRE,
                                new DamageRange(80.0, 120.0),
                                1.0,  // ability power scaling
                                null  // no area
                            ),
                            null,  // no status effect
                            null,  // no duration
                            List.of()  // no modifiers
                        )
                    )
                )
            ),
            "Hurls a ball of fire at the target.",
            "fireball_icon.png"
        );
    }
    
    @Override
    public CompletableFuture<AbilityResult> execute(CombatContext context) {
        CombatEntity caster = context.caster();
        CombatEntity target = context.target();
        
        // Get ability power
        double abilityPower = caster.getStats().get(CombatStats.ABILITY_POWER);
        
        // Calculate damage
        AbilityLevel level = getDefinition().levels().get(0);
        DamageDefinition damageDef = level.effects().get(0).damage();
        double baseDamage = damageDef.damageRange().min() 
            + Math.random() * (damageDef.damageRange().max() - damageDef.damageRange().min());
        double scaledDamage = baseDamage + (abilityPower * damageDef.abilityPowerScaling());
        
        // Create damage event
        DamageEvent event = new DamageEvent(
            UUID.randomUUID(),
            caster,
            target,
            Map.of(DamageType.FIRE, scaledDamage),
            false,  // not attack
            true,   // is spell
            true,   // is projectile
            false,  // not area
            true,   // can crit
            true,   // can dodge
            false,  // can't block spells
            "fireball",
            Instant.now()
        );
        
        // Deal damage
        DamageResult damageResult = combatService.dealDamage(event);
        
        // Build result
        return CompletableFuture.completedFuture(
            new AbilityResult(
                UUID.randomUUID(),
                true,  // success
                null,  // no failure reason
                1,     // cast count
                List.of(event),  // damage events
                List.of(),       // no heal events
                List.of(),       // no effects applied
                Instant.now()
            )
        );
    }
}
```

### Custom Effect with Periodic Damage

```java
public ActiveEffect createBurnEffect(CombatEntity source, double damagePerTick, Duration duration) {
    // Periodic damage component
    PeriodicEffect periodic = new PeriodicEffect(
        DamageType.FIRE,
        damagePerTick,
        Duration.ofSeconds(1),  // tick every 1 second
        Instant.now().plusSeconds(1)  // first tick in 1 second
    );
    
    return new ActiveEffect(
        "burn_" + UUID.randomUUID(),
        source.toString(),  // source ID
        EffectCategory.DEBUFF,
        List.of(),  // no stat modifiers
        Instant.now(),
        Instant.now().plus(duration),
        1,
        1,
        periodic,  // periodic effect
        "burn_icon.png",
        true
    );
}

// Apply and tick
effectService.applyEffect(target, createBurnEffect(attacker, 10.0, Duration.ofSeconds(10)));

// In server tick loop:
effectService.tickPeriodicEffects(target);
```

### Stat Snapshots

Capture stats at a point in time:

```java
public record StatSnapshot(
    double strength,
    double agility,
    double physicalPower,
    double critChance,
    Instant timestamp
) {
    public static StatSnapshot capture(CombatEntity entity) {
        StatContainer stats = entity.getStats();
        return new StatSnapshot(
            stats.get(CombatStats.STRENGTH),
            stats.get(CombatStats.AGILITY),
            stats.get(CombatStats.PHYSICAL_POWER),
            stats.get(CombatStats.CRITICAL_CHANCE),
            Instant.now()
        );
    }
}

// Usage
StatSnapshot before = StatSnapshot.capture(player);
applyBuff(player);
StatSnapshot after = StatSnapshot.capture(player);

logger.info("Buff increased physical power by {}", 
    after.physicalPower() - before.physicalPower()
);
```

---

## Troubleshooting

### Issue: "Cannot find symbol: getRawDamage()"

**Cause**: Using Lombok getter syntax on Java 25 records.

**Fix**: Use record accessor syntax:
```java
// ❌ WRONG
double damage = event.getRawDamage();

// ✅ CORRECT
double damage = event.rawDamage();
```

### Issue: "NoSuchMethodError: builder()"

**Cause**: Trying to use Lombok builder pattern on records.

**Fix**: Use record constructor or factory methods:
```java
// ❌ WRONG
DamageResult result = DamageResult.builder()
    .eventId(id)
    .totalDamage(100.0)
    .build();

// ✅ CORRECT
DamageResult result = new DamageResult(
    id,
    rawDamage,
    mitigatedDamage,
    0.0,  // dodgeReduction
    0.0,  // blockReduction
    finalDamage,
    100.0,  // totalDamage
    false,  // wasCritical
    false,  // wasDodged
    false,  // wasBlocked
    List.of(),  // triggeredEffects
    Instant.now()
);
```

### Issue: Stats not updating after adding modifier

**Cause**: Forgot to recalculate derived stats.

**Fix**: Call `recalculateStats()` after adding modifiers:
```java
stats.addModifier(modifier);
entity.recalculateStats();  // ← Important!
```

### Issue: Effects not applying

**Cause**: Effect modifiers have wrong source ID.

**Fix**: Ensure unique source IDs and use `removeModifiersBySource()`:
```java
// Each effect should have unique source
ActiveEffect effect = new ActiveEffect(
    "buff_123",
    "buff:battle_cry",  // ← Source ID for cleanup
    // ...
);

// Remove by source when effect ends
stats.removeModifiersBySource("buff:battle_cry");
```

### Issue: "Class not found" when injecting service

**Cause**: CombatModule not installed in Guice injector.

**Fix**: Install module in your mod setup:
```java
Injector injector = Guice.createInjector(
    new CombatModule(),  // ← Add this
    new MyModModule()
);
```

---

## Next Steps

- **API Reference**: See [[api/framework-combat]] for complete API documentation
- **Examples**: Check `00-Argonath-Samples/combat-examples/` for more code samples
- **Specs**: Read [[SF-14-B-combat-framework]] for implementation details
- **Discord**: Join #combat-framework for support

---

**Last Updated**: 2026-01-27  
**Framework Version**: 2.0.0+  
**Status**: Production Ready
