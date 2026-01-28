# Combat Framework API Reference

> **Package**: `com.argonathsystems.combat.*`  
> **Framework**: `04-framework-combat`  
> **Version**: 2.0.0+  
> **Language**: Java 25  
> **Last Updated**: 2026-01-27

---

## Table of Contents

1. [Service Layer](#service-layer)
2. [Domain Models](#domain-models)
3. [Combat Stats](#combat-stats)
4. [Abilities](#abilities)
5. [Effects](#effects)
6. [Equipment](#equipment)
7. [Migration Notes](#migration-notes)

---

## Service Layer

### CombatService

**Interface**: `com.argonathsystems.combat.service.CombatService`

Core combat calculation and execution service.

#### Methods

```java
/**
 * Calculate and apply damage to a target.
 *
 * @param event The damage event containing attacker, target, and damage details
 * @return DamageResult with final damage, mitigation, and flags
 */
DamageResult dealDamage(DamageEvent event);

/**
 * Execute an ability.
 *
 * @param caster The entity casting the ability
 * @param abilityId The ID of the ability to execute
 * @param target The ability's target(s)
 * @return CompletableFuture with AbilityResult (async execution)
 */
CompletableFuture<AbilityResult> executeAbility(
    CombatEntity caster,
    String abilityId,
    AbilityTarget target
);

/**
 * Check if an ability is on cooldown.
 *
 * @param entity The entity to check
 * @param abilityId The ability ID
 * @return true if on cooldown, false if ready
 */
boolean isAbilityOnCooldown(CombatEntity entity, String abilityId);

/**
 * Get remaining cooldown time.
 *
 * @param entity The entity to check
 * @param abilityId The ability ID
 * @return Duration remaining, or Duration.ZERO if ready
 */
Duration getRemainingCooldown(CombatEntity entity, String abilityId);
```

**Implementation**: `com.argonathsystems.combat.service.impl.CombatServiceImpl`

**Injection**:
```java
@Inject
public MyClass(CombatService combatService) {
    this.combatService = combatService;
}
```

---

### EffectService

**Interface**: `com.argonathsystems.combat.service.EffectService`

Manages status effects (buffs, debuffs, DoTs, HoTs).

#### Methods

```java
/**
 * Apply an effect to an entity.
 * Automatically integrates modifiers with StatContainer.
 *
 * @param target The entity to apply the effect to
 * @param effect The effect to apply
 */
void applyEffect(CombatEntity target, ActiveEffect effect);

/**
 * Remove an effect by ID.
 *
 * @param target The entity to remove the effect from
 * @param effectId The effect ID
 */
void removeEffect(CombatEntity target, String effectId);

/**
 * Tick periodic effects (DoT/HoT).
 * Call once per server tick for all active entities.
 *
 * @param entity The entity to tick
 */
void tickPeriodicEffects(CombatEntity entity);

/**
 * Remove expired effects.
 * Call periodically to clean up.
 *
 * @param entity The entity to clean
 */
void cleanupExpiredEffects(CombatEntity entity);

/**
 * Get all active effects of a category.
 *
 * @param entity The entity to query
 * @param category The effect category (BUFF, DEBUFF, etc.)
 * @return List of active effects in category
 */
List<ActiveEffect> getActiveEffectsByCategory(
    CombatEntity entity, 
    EffectCategory category
);
```

**Implementation**: `com.argonathsystems.combat.service.impl.EffectServiceImpl`

---

### GemService

**Interface**: `com.argonathsystems.combat.service.GemService`

Manages skill gem socketing and linking.

#### Methods

```java
/**
 * Socket a gem into equipment.
 *
 * @param equipment The equipment to socket
 * @param socketIndex The socket index (0-based)
 * @param gem The gem to insert
 * @return true if successful, false if socket occupied or invalid
 */
boolean socketGem(Equipment equipment, int socketIndex, SkillGem gem);

/**
 * Remove a gem from a socket.
 *
 * @param equipment The equipment
 * @param socketIndex The socket index
 * @return The removed gem, or null if socket was empty
 */
SkillGem unsocketGem(Equipment equipment, int socketIndex);

/**
 * Get all socketed gems in equipment.
 *
 * @param equipment The equipment
 * @return List of gems (nulls for empty sockets)
 */
List<SkillGem> getSocketedGems(Equipment equipment);

/**
 * Check if gem can be socketed (color matching).
 *
 * @param socket The socket
 * @param gem The gem
 * @return true if compatible
 */
boolean canSocketGem(GemSocket socket, SkillGem gem);
```

**Implementation**: `com.argonathsystems.combat.service.impl.GemServiceImpl`

---

## Domain Models

All domain models are **Java 25 records** (immutable by default).

### DamageEvent

**Package**: `com.argonathsystems.combat.domain.damage`

Represents a damage event before mitigation.

```java
public record DamageEvent(
    UUID eventId,
    CombatEntity attacker,
    CombatEntity target,
    Map<DamageType, Double> rawDamage,
    boolean isAttack,
    boolean isSpell,
    boolean isProjectile,
    boolean isArea,
    boolean canCrit,
    boolean canDodge,
    boolean canBlock,
    String abilityId,
    Instant timestamp
) {
    // Factory method
    public static DamageEvent create(
        CombatEntity attacker,
        CombatEntity target,
        Map<DamageType, Double> rawDamage
    ) {
        return new DamageEvent(
            UUID.randomUUID(),
            attacker,
            target,
            rawDamage,
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
    }
}
```

**Accessors**: Use record field names (NO `get` prefix)
- `event.eventId()` ✅ NOT ~~`event.getEventId()`~~ ❌
- `event.rawDamage()` ✅ NOT ~~`event.getRawDamage()`~~ ❌
- `event.canDodge()` ✅ NOT ~~`event.isCanDodge()`~~ ❌

---

### DamageResult

**Package**: `com.argonathsystems.combat.domain.damage`

Result of damage calculation with mitigation.

```java
public record DamageResult(
    UUID eventId,
    Map<DamageType, Double> rawDamage,
    Map<DamageType, Double> mitigatedDamage,
    double dodgeReduction,
    double blockReduction,
    double finalDamage,
    double totalDamage,
    boolean wasCritical,
    boolean wasDodged,
    boolean wasBlocked,
    List<String> triggeredEffects,
    Instant timestamp
) {}
```

**Usage**:
```java
DamageResult result = combatService.dealDamage(event);

if (result.wasCritical()) {
    logger.info("Critical hit for {} damage!", result.totalDamage());
}

if (result.wasDodged()) {
    logger.info("Attack dodged!");
}
```

---

### ActiveEffect

**Package**: `com.argonathsystems.combat.domain.effect`

Active status effect on an entity.

```java
public record ActiveEffect(
    String effectId,
    String sourceId,
    EffectCategory category,
    List<Modifier> modifiers,
    Instant appliedAt,
    Instant expiresAt,
    int stacks,
    int maxStacks,
    PeriodicEffect periodicEffect,
    String iconPath,
    boolean showInUI
) {
    // Check if effect has expired
    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }
    
    // Get remaining duration
    public Duration getRemainingDuration() {
        return Duration.between(Instant.now(), expiresAt);
    }
}
```

**Categories**:
- `BUFF` - Positive effect
- `DEBUFF` - Negative effect
- `NEUTRAL` - Neither positive nor negative

---

### AbilityDefinition

**Package**: `com.argonathsystems.combat.domain.ability`

Defines an ability's static properties.

```java
public record AbilityDefinition(
    String id,
    String name,
    AbilityType type,
    List<AbilityLevel> levels,
    String description,
    String iconPath
) {}
```

**Ability Types**:
- `MELEE_ATTACK` - Physical melee
- `RANGED_ATTACK` - Physical ranged
- `SPELL` - Magic spell
- `SUPPORT` - Buff/heal
- `PASSIVE` - Always active

---

### AbilityResult

**Package**: `com.argonathsystems.combat.domain.ability`

Result of ability execution.

```java
public record AbilityResult(
    UUID executionId,
    boolean success,
    String failureReason,
    int castCount,
    List<DamageEvent> damageEvents,
    List<HealEvent> healEvents,
    List<ActiveEffect> effectsApplied,
    Instant timestamp
) {}
```

**Usage**:
```java
CompletableFuture<AbilityResult> future = combatService.executeAbility(
    caster, 
    "fireball", 
    AbilityTarget.single(target)
);

future.thenAccept(result -> {
    if (result.success()) {
        logger.info("Ability hit {} targets", result.damageEvents().size());
    } else {
        logger.warn("Ability failed: {}", result.failureReason());
    }
});
```

---

## Combat Stats

**Package**: `com.argonathsystems.combat.domain.stats`

All combat stats are defined as `StatId` constants in `CombatStats`.

### Primary Attributes

```java
public static final StatId STRENGTH = new StatId("combat", "strength");
public static final StatId AGILITY = new StatId("combat", "agility");
public static final StatId INTELLIGENCE = new StatId("combat", "intelligence");
public static final StatId VITALITY = new StatId("combat", "vitality");
public static final StatId ENDURANCE = new StatId("combat", "endurance");
public static final StatId LUCK = new StatId("combat", "luck");
```

### Derived Combat Stats

```java
// Power
public static final StatId PHYSICAL_POWER = new StatId("combat", "physical_power");
public static final StatId ABILITY_POWER = new StatId("combat", "ability_power");

// Critical
public static final StatId CRITICAL_CHANCE = new StatId("combat", "critical_chance");
public static final StatId CRITICAL_DAMAGE = new StatId("combat", "critical_damage");

// Attack Speed
public static final StatId ATTACK_SPEED = new StatId("combat", "attack_speed");
public static final StatId CAST_SPEED = new StatId("combat", "cast_speed");

// Defense
public static final StatId ARMOR = new StatId("combat", "armor");
public static final StatId MAGIC_RESIST = new StatId("combat", "magic_resist");
public static final StatId DODGE_CHANCE = new StatId("combat", "dodge_chance");
public static final StatId BLOCK_CHANCE = new StatId("combat", "block_chance");
```

### Health & Resources

```java
public static final StatId CURRENT_HEALTH = new StatId("combat", "current_health");
public static final StatId MAX_HEALTH = new StatId("combat", "max_health");
public static final StatId HEALTH_REGEN = new StatId("combat", "health_regen");

public static final StatId CURRENT_MANA = new StatId("combat", "current_mana");
public static final StatId MAX_MANA = new StatId("combat", "max_mana");
public static final StatId MANA_REGEN = new StatId("combat", "mana_regen");

public static final StatId CURRENT_STAMINA = new StatId("combat", "current_stamina");
public static final StatId MAX_STAMINA = new StatId("combat", "max_stamina");
public static final StatId STAMINA_REGEN = new StatId("combat", "stamina_regen");
```

### Utility Methods

```java
/**
 * Initialize entity with default combat stats.
 */
public static void initializeDefaults(StatContainer stats) {
    stats.set(STRENGTH, 10.0);
    stats.set(AGILITY, 10.0);
    stats.set(INTELLIGENCE, 10.0);
    // ... all base stats
}

/**
 * Calculate derived stats from primary attributes.
 */
public static void calculateDerivedStats(StatContainer stats) {
    // Physical Power = Strength * 2
    stats.set(PHYSICAL_POWER, stats.get(STRENGTH) * 2.0);
    
    // Critical Chance = Agility * 0.1% (max 50%)
    double critChance = Math.min(stats.get(AGILITY) * 0.001, 0.50);
    stats.set(CRITICAL_CHANCE, critChance);
    
    // Max Health = Vitality * 20 + 100
    stats.set(MAX_HEALTH, stats.get(VITALITY) * 20.0 + 100.0);
    
    // ... all derived stats
}
```

**Usage**:
```java
StatContainer stats = entity.getStats();

// Get stat value (includes modifiers)
double strength = stats.get(CombatStats.STRENGTH);

// Add modifier
stats.addModifier(new FlatModifier(
    UUID.randomUUID(),
    CombatStats.STRENGTH,
    "Sword of Power",
    "equipment:main_hand",
    50.0
));

// Recalculate derived stats
CombatStats.calculateDerivedStats(stats);
```

---

## Abilities

### Ability Interface

**Interface**: `com.argonathsystems.combat.domain.ability.Ability`

Implement this interface to create custom abilities.

```java
public interface Ability {
    /**
     * Get ability's static definition.
     */
    AbilityDefinition getDefinition();
    
    /**
     * Execute the ability.
     *
     * @param context Execution context (caster, target, level, etc.)
     * @return CompletableFuture with execution result
     */
    CompletableFuture<AbilityResult> execute(CombatContext context);
}
```

### CombatContext

**Record**: `com.argonathsystems.combat.domain.ability.CombatContext`

Context passed to ability execution.

```java
public record CombatContext(
    CombatEntity caster,
    AbilityTarget target,
    int abilityLevel,
    Map<String, Object> parameters
) {}
```

### AbilityTarget

**Record**: `com.argonathsystems.combat.domain.ability.AbilityTarget`

Defines ability targets.

```java
public record AbilityTarget(
    List<CombatEntity> entities,
    Optional<Vector3> location
) {
    public static AbilityTarget single(CombatEntity entity) {
        return new AbilityTarget(List.of(entity), Optional.empty());
    }
    
    public static AbilityTarget multiple(List<CombatEntity> entities) {
        return new AbilityTarget(entities, Optional.empty());
    }
    
    public static AbilityTarget location(Vector3 location) {
        return new AbilityTarget(List.of(), Optional.of(location));
    }
}
```

---

## Effects

### PeriodicEffect

**Record**: `com.argonathsystems.combat.domain.effect.PeriodicEffect`

Periodic damage or healing.

```java
public record PeriodicEffect(
    DamageType damageType,
    double damagePerTick,
    Duration tickInterval,
    Instant nextTickAt
) {}
```

**Example**:
```java
// Poison: 5 damage per second for 10 seconds
PeriodicEffect poison = new PeriodicEffect(
    DamageType.POISON,
    5.0,
    Duration.ofSeconds(1),
    Instant.now().plusSeconds(1)
);

ActiveEffect poisonEffect = new ActiveEffect(
    "poison_" + UUID.randomUUID(),
    caster.toString(),
    EffectCategory.DEBUFF,
    List.of(),  // no stat modifiers
    Instant.now(),
    Instant.now().plusSeconds(10),
    1,
    1,
    poison,  // periodic effect
    "poison_icon.png",
    true
);
```

---

## Equipment

### EquipmentSnapshot

**Record**: `com.argonathsystems.combat.domain.equipment.EquipmentSnapshot`

Snapshot of entity's equipment state.

```java
public record EquipmentSnapshot(
    Equipment mainHand,
    Equipment offHand,
    Equipment head,
    Equipment chest,
    Equipment legs,
    Equipment feet,
    Equipment accessory1,
    Equipment accessory2
) {}
```

### Equipment

**Record**: `com.argonathsystems.combat.domain.equipment.Equipment`

Single equipment piece.

```java
public record Equipment(
    String itemId,
    String name,
    EquipmentSlot slot,
    List<Modifier> statModifiers,
    List<GemSocket> gemSockets,
    int durability,
    int maxDurability
) {}
```

### GemSocket

**Record**: `com.argonathsystems.combat.domain.equipment.GemSocket`

Socket for skill gems.

```java
public record GemSocket(
    int socketIndex,
    SocketColor color,
    boolean isLinked,
    SkillGem socketedGem
) {}
```

**Socket Colors**:
- `RED` - Strength gems
- `GREEN` - Agility gems
- `BLUE` - Intelligence gems
- `WHITE` - Universal (any color)

---

## Migration Notes

### From Lombok to Records

#### Getter Syntax

```java
// ❌ OLD (Lombok)
double damage = event.getRawDamage();
String id = event.getAbilityId();
boolean crit = event.isCanCrit();

// ✅ NEW (Records)
double damage = event.rawDamage();
String id = event.abilityId();
boolean crit = event.canCrit();
```

#### Builder Pattern

```java
// ❌ OLD (Lombok)
DamageEvent event = DamageEvent.builder()
    .eventId(UUID.randomUUID())
    .attacker(attacker)
    .target(target)
    .build();

// ✅ NEW (Factory Methods)
DamageEvent event = DamageEvent.create(attacker, target, damageMap);

// OR (Full Constructor)
DamageEvent event = new DamageEvent(
    UUID.randomUUID(),
    attacker,
    target,
    damageMap,
    true, false, false, false,
    true, true, true,
    null,
    Instant.now()
);
```

#### Null Safety

Records require non-null values (unless explicitly nullable).

```java
// ❌ OLD
@Builder
public class DamageEvent {
    private String abilityId;  // Can be null
}

// ✅ NEW
public record DamageEvent(
    String abilityId  // Still can be null, but explicit
) {
    // Add validation in compact constructor if needed
    public DamageEvent {
        // abilityId can be null for basic attacks
    }
}
```

### From Old Stat System to Stats Framework

#### Getting Stats

```java
// ❌ OLD
double strength = entity.getStrength();
double critChance = entity.getCriticalChance();

// ✅ NEW
StatContainer stats = entity.getStats();
double strength = stats.get(CombatStats.STRENGTH);
double critChance = stats.get(CombatStats.CRITICAL_CHANCE);
```

#### Applying Buffs

```java
// ❌ OLD
entity.addBuff(new StrengthBuff(50.0, Duration.ofSeconds(10)));

// ✅ NEW
List<Modifier> modifiers = List.of(
    new FlatModifier(
        UUID.randomUUID(),
        CombatStats.STRENGTH,
        "Strength Potion",
        "buff:strength_potion",
        50.0
    )
);

ActiveEffect buff = new ActiveEffect(
    "buff_" + UUID.randomUUID(),
    "potion:strength",
    EffectCategory.BUFF,
    modifiers,
    Instant.now(),
    Instant.now().plusSeconds(10),
    1, 1, null,
    "strength_icon.png",
    true
);

effectService.applyEffect(entity, buff);
```

---

## Code Examples

### Complete Combat Flow

```java
public class CombatExample {
    @Inject private CombatService combatService;
    @Inject private EffectService effectService;
    
    public void performAttack(CombatEntity attacker, CombatEntity target) {
        // 1. Get attacker's stats
        StatContainer stats = attacker.getStats();
        double physicalPower = stats.get(CombatStats.PHYSICAL_POWER);
        double critChance = stats.get(CombatStats.CRITICAL_CHANCE);
        
        // 2. Create damage event
        DamageEvent event = new DamageEvent(
            UUID.randomUUID(),
            attacker,
            target,
            Map.of(DamageType.PHYSICAL, physicalPower),
            true, false, false, false,
            true, true, true,
            null,
            Instant.now()
        );
        
        // 3. Deal damage
        DamageResult result = combatService.dealDamage(event);
        
        // 4. Apply on-hit effects
        if (result.wasCritical() && Math.random() < 0.20) {
            ActiveEffect bleed = createBleedEffect(attacker, 10.0, Duration.ofSeconds(5));
            effectService.applyEffect(target, bleed);
        }
        
        // 5. Log result
        logger.info("Dealt {} damage (crit: {}, dodged: {})",
            result.totalDamage(),
            result.wasCritical(),
            result.wasDodged()
        );
    }
    
    private ActiveEffect createBleedEffect(CombatEntity source, double dps, Duration duration) {
        PeriodicEffect periodic = new PeriodicEffect(
            DamageType.PHYSICAL,
            dps,
            Duration.ofSeconds(1),
            Instant.now().plusSeconds(1)
        );
        
        return new ActiveEffect(
            "bleed_" + UUID.randomUUID(),
            source.toString(),
            EffectCategory.DEBUFF,
            List.of(),
            Instant.now(),
            Instant.now().plus(duration),
            1, 1,
            periodic,
            "bleed_icon.png",
            true
        );
    }
}
```

---

## Related Documentation

- **User Guide**: See [[guides/combat-framework-usage]] for practical examples
- **Specification**: See [[00-Argonath-Specifications/07-Combat/SF-14-B-combat-framework]] for design details
- **Stats Framework**: See [[api/framework-stats]] for stat system API
- **Examples**: Check `00-Argonath-Samples/combat-examples/`

---

**Last Updated**: 2026-01-27  
**Framework Version**: 2.0.0+  
**Status**: Production Ready
