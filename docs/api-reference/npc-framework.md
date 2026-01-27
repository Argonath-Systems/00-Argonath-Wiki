# NPC Framework API Reference

The NPC Framework provides a comprehensive, platform-agnostic system for creating and managing NPCs in the Argonath Systems game server.

## Overview

The framework follows the **Accessor Pattern** to maintain platform independence. All Hytale-specific code is isolated in the adapter layer.

```
04-framework-npc/
├── api/               # Public API (NPCDefinition, builders)
├── behavior/          # Sound, movement, animation behaviors
├── config/            # Configuration loading
├── event/             # NPC lifecycle events
├── loot/              # Loot table integration
├── narrative/         # Name generation, personality traits
├── origin/            # Multi-dimensional origin system
└── spawn/             # Spawn and appearance configuration
```

## Quick Start

### Creating an NPC Definition

```java
NPCDefinition blacksmith = NPCDefinitionBuilder.create("blacksmith_thorin")
    .displayName("Thorin Ironforge")
    .profession("blacksmith")
    .wealthLevel(3)
    .backstory("A skilled dwarven craftsman...")
    
    // Multi-layered origin
    .origin(CompositeOrigin.builder()
        .geographic("city", "Major City")
        .cultural("dwarven", "Dwarven Kingdoms")
        .social("artisan", "Skilled Artisan")
        .build())
    
    // Personality traits
    .addPersonalityTrait("hardworking")
    .addPersonalityTrait("perfectionist")
    
    // Appearance
    .appearance(NPCAppearance.builder()
        .entityType("hytale:dwarf")
        .skin("skins/dwarf_blacksmith")
        .mainHand("item:blacksmith_hammer")
        .scale(0.9f)
        .build())
    
    // Behaviors
    .soundBehavior(NPCSoundBehavior.builder()
        .onInteract("npc:dwarf_greet")
        .onDeath("npc:dwarf_death")
        .build())
    .movementBehavior(NPCMovementBehavior.walking())
    .animationBehavior(NPCAnimationBehavior.humanoid())
    
    // Loot
    .lootTable("loot:dwarf_blacksmith")
    
    // Spawn location
    .spawn(NPCSpawnConfig.at("world", 100, 64, 200))
    .build();
```

---

## Core API

### NPCDefinition

The unified interface representing a complete NPC template.

```java
public interface NPCDefinition {
    String getTemplateId();
    String getDisplayName();
    String getProfession();
    int getWealthLevel();
    
    Optional<CompositeOrigin> getOrigin();
    Set<String> getPersonalityTraits();
    Optional<NPCAppearance> getAppearance();
    Optional<NPCSoundBehavior> getSoundBehavior();
    Optional<NPCMovementBehavior> getMovementBehavior();
    Optional<NPCAnimationBehavior> getAnimationBehavior();
    Optional<NPCLootConfig> getLootConfig();
    Optional<NPCSpawnConfig> getSpawnConfig();
    
    Optional<String> getBackstory();
    Optional<String> getDialogTreePath();
    boolean isInvulnerable();
    boolean isInteractable();
}
```

### NPCDefinitionBuilder

Fluent builder for creating NPC definitions.

```java
NPCDefinitionBuilder builder = NPCDefinitionBuilder.create("npc_id")
    .displayName("NPC Name")
    .profession("merchant")
    .wealthLevel(2)  // 0-5 scale
    .invulnerable(false)
    .interactable(true);
```

---

## Origin System

### Multi-Dimensional Origins

NPCs can have origins across multiple dimensions:

| Dimension | Examples |
|-----------|----------|
| Geographic | village, city, rural, nomadic, foreign |
| Cultural | francia, roman, byzantine, persian |
| Social | peasant, artisan, merchant, noble |
| Religious | devout, secular, clergy |
| Occupational | farmer, smith, scholar, soldier |

### CompositeOrigin

```java
CompositeOrigin origin = CompositeOrigin.builder()
    .geographic("village", "Small Village")
    .cultural("francia", "Frankish Kingdom")
    .social("peasant", "Peasant Class")
    .religious("devout", "Devout Believer")
    .build();

// Query dimensions
origin.getGeographic();  // Optional<OriginDimension>
origin.getCultural();    // Optional<OriginDimension>

// Generate narrative
String narrative = origin.toNarrativeString();
// "a peasant from Small Village in Frankish Kingdom"
```

### OriginRegistry

Configure available origin values:

```java
OriginRegistry registry = OriginRegistry.withDefaultTypes();
registry.registerDimension("geographic", 
    new OriginDimension("castle", "Castle", "A fortified stronghold"));

// Or load from config
OriginRegistry registry = configLoader.loadOriginRegistry();
```

---

## Behavior System

### NPCSoundBehavior

Event-driven sound configuration:

```java
NPCSoundBehavior sounds = NPCSoundBehavior.builder()
    .onInteract("npc:greet")
    .onHit("npc:pain")
    .onDeath("npc:death")
    .onKill("npc:victory")
    .onSpawn("npc:spawn")
    .volume(0.8f)
    .pitchVariation(0.2f)
    
    // Ambient sounds
    .addAmbient("idle", "npc:idle_hum", 0.3f, 30, 120)
    .build();
```

### NPCMovementBehavior

```java
NPCMovementBehavior movement = NPCMovementBehavior.builder()
    .style(MovementStyle.WALK)
    .baseSpeed(1.0f)
    .canSprint(true)
    .canCrouch(false)
    
    // Wandering behavior
    .wander(20.0f, 5, 30, true)  // radius, minWait, maxWait, stayInArea
    
    // Or patrol path
    .patrol("path:market_square")
    .build();

// Presets
NPCMovementBehavior.stationary();
NPCMovementBehavior.walking();
```

### NPCAnimationBehavior

```java
NPCAnimationBehavior animations = NPCAnimationBehavior.builder()
    .idle("anim:idle")
    .walk("anim:walk")
    .run("anim:run")
    .interact("anim:greet")
    .attack("anim:attack")
    .death("anim:death")
    
    // Custom animations
    .wave("anim:wave")
    .fidget("anim:fidget", 0.3f, 60)  // 30% chance, 60s cooldown
    .build();

// Preset
NPCAnimationBehavior.humanoid();
```

---

## Loot System

### NPCLootConfig

```java
NPCLootConfig loot = NPCLootConfig.builder()
    .lootTable("loot:goblin_warrior")
    .dropRateModifier(1.5f)
    .luckInfluence(0.8f)
    
    // Guaranteed drops
    .addGuaranteedDrop("item:goblin_ear", 1)
    .addGuaranteedDrop("item:gold_coin", 5, 10)  // 5-10 coins
    
    // Chance drops
    .addChanceDrop("item:rare_gem", 1, 0.05f)  // 5% chance
    
    .dropsOnDeath(true)
    .dropsOnPickpocket(false)
    .build();

// Shorthand
NPCLootConfig.fromTable("loot:skeleton");
NPCLootConfig.none();
```

### NPCLootService

Automatically drops loot on NPC death events:

```java
NPCLootService lootService = new NPCLootService(
    lootTableAccessor,
    eventAccessor,
    npcId -> lootConfigs.get(npcId),
    npcId -> wealthLevels.get(npcId)
);

lootService.start();  // Begin listening to death events
```

---

## Event System

### Type-Safe Events

All events in the accessor framework must implement `AccessorEvent` for compile-time type safety:

```java
// AccessorEvent - base interface for all events
public interface AccessorEvent {
    default String getEventName();
    default long getTimestamp();
}

// CancellableEvent - for events that can be cancelled
public interface CancellableEvent extends AccessorEvent {
    boolean isCancelled();
    void setCancelled(boolean cancelled);
}

// NPCEvent - base for all NPC events
public interface NPCEvent extends AccessorEvent {
    String getNpcId();
}
```

This ensures that only proper event types can be registered:

```java
// ✅ Compiles - NPCDeathEvent extends NPCEvent extends AccessorEvent
eventAccessor.register(NPCDeathEvent.class, event -> { ... });

// ❌ Won't compile - String doesn't extend AccessorEvent
eventAccessor.register(String.class, s -> { ... });
```

### NPC Events

| Event | Trigger |
|-------|---------|
| `NPCSpawnEvent` | NPC spawns in world |
| `NPCInteractEvent` | Player interacts with NPC |
| `NPCDamageEvent` | NPC takes damage |
| `NPCDeathEvent` | NPC dies |
| `NPCKillEvent` | NPC kills something |

### Event Usage

```java
// Subscribe to events
eventAccessor.register(NPCDeathEvent.class, event -> {
    System.out.println("NPC " + event.npcId() + " died at " + 
        event.x() + ", " + event.y() + ", " + event.z());
    
    if (event.wasKilledByPlayer()) {
        UUID killer = event.killerId();
        // Award XP, update quests, etc.
    }
});

// Event data
NPCInteractEvent event = ...;
event.npcId();           // String
event.playerId();        // UUID
event.interactionType(); // CLICK, APPROACH, TRADE, DIALOGUE, etc.
```

---

## Personality Traits

### WeightedPersonalityTrait

Traits with weights for random selection:

```java
WeightedPersonalityTrait trait = new WeightedPersonalityTrait(
    "ambitious",
    "Ambitious",
    "Driven to succeed",
    TraitCategory.ECONOMIC,
    1.0f,              // Base weight
    new String[]{}     // Conflicting traits
);

// With conflicts
WeightedPersonalityTrait greedy = WeightedPersonalityTrait.weighted(
    "greedy", "Greedy", TraitCategory.ECONOMIC, 0.6f
).withConflicts("generous");
```

### PersonalityTraitRegistry

```java
PersonalityTraitRegistry registry = PersonalityTraitRegistry.withDefaults();

// Select random traits
Set<String> traits = registry.selectTraits(
    TraitSelectionConfig.builder()
        .minTraits(2)
        .maxTraits(4)
        .maxPerCategory(1)
        .excludeTrait("greedy")
        .build()
);
// Returns e.g. {"brave", "curious", "hardworking"}
```

---

## Spawn & Appearance

### NPCAppearance

```java
NPCAppearance appearance = NPCAppearance.builder()
    .entityType("hytale:human")
    .skin("skins/blacksmith_male")
    .modelVariant("burly")
    
    // Equipment
    .mainHand("item:hammer")
    .offHand("item:tongs")
    .helmet("item:leather_cap")
    .chestplate("item:leather_apron")
    
    // Visual effects
    .scale(1.1f)
    .glowing(false)
    .nametagColor("§6")
    
    // Custom data
    .customData("beard_style", "long")
    .build();

// Shorthand
NPCAppearance.humanoid("skins/villager");
```

### NPCSpawnConfig

```java
NPCSpawnConfig spawn = NPCSpawnConfig.builder()
    .world("overworld")
    .position(100.5, 64.0, -200.5)
    .rotation(90.0f, 0.0f)  // yaw, pitch
    .spawnType(SpawnType.RESPAWNING)
    .respawnDelay(600)  // seconds
    .persistent(true)
    .spawnGroup("market_vendors")
    .build();

// Shorthand
NPCSpawnConfig.at("world", 100, 64, 200);
```

---

## Configuration Loading

### Directory Structure

NPCs can be loaded from a directory of YAML or JSON files:

```
config/npcs/
├── blacksmith.yml          # Single NPC definition
├── merchants/
│   ├── trader_marcus.yml
│   └── vendor_elara.yml
└── guards/
    ├── city_guard.yml
    └── night_watch.yml
```

### NPCConfigStore - CRUD Operations

The `NPCConfigStore` provides full CRUD operations for NPC configurations:

```java
// Initialize the store
Path npcConfigDir = Path.of("config/npcs");
NPCConfigStore store = new NPCConfigStore(npcConfigDir);

// Load all NPCs from directory
Map<String, NPCDefinition> allNPCs = store.loadAll();

// Load a specific NPC
Optional<NPCDefinition> blacksmith = store.load("blacksmith");

// Save a new NPC
NPCDefinition newNPC = NPCDefinitionBuilder.create("new_merchant")
    .displayName("New Merchant")
    .profession("merchant")
    .build();
store.save(newNPC);

// Save to a subdirectory
store.save(newNPC, "merchants");

// Update an existing NPC
NPCDefinition updated = NPCDefinitionBuilder.create("blacksmith")
    .displayName("Updated Blacksmith")
    .profession("blacksmith")
    .wealthLevel(4)
    .build();
store.update(updated);

// Delete an NPC
boolean deleted = store.delete("old_npc");

// Hot reload a specific NPC
store.reload("blacksmith");

// Reload all NPCs
store.reloadAll();
```

### Query Methods

```java
// Get all loaded NPCs
Collection<NPCDefinition> all = store.getAll();

// Get all template IDs
Set<String> ids = store.getAllIds();

// Get NPCs by profession
List<NPCDefinition> merchants = store.getByProfession("merchant");

// Get NPCs by faction
List<NPCDefinition> gondorNPCs = store.getByFaction("gondor");

// Check if NPC exists
boolean exists = store.exists("blacksmith");
```

### NPCDefinitionSerializer

Convert NPCDefinitions back to configuration maps for saving:

```java
NPCDefinitionSerializer serializer = new NPCDefinitionSerializer();

// Serialize to Map (can be written as YAML or JSON)
Map<String, Object> configData = serializer.serialize(npcDefinition);
```

### YAML Configuration

**origins.yml**:
```yaml
dimension_types:
  geographic:
    display_name: "Geographic Origin"
    priority: 100
    required: true
    values:
      - id: village
        display_name: "Village"
        description: "A small rural settlement"
      - id: city
        display_name: "City"
        description: "A major urban center"
```

**personality-traits.yml**:
```yaml
categories:
  economic:
    traits:
      - id: greedy
        weight: 0.6
        conflicts: [generous]
      - id: generous
        weight: 0.8
        conflicts: [greedy]
```

### Loading Configuration

```java
NPCConfigLoader loader = new NPCConfigLoader(originConfig, traitConfig);
OriginRegistry origins = loader.loadOriginRegistry();
PersonalityTraitRegistry traits = loader.loadPersonalityTraitRegistry();
```

### Parsing NPC Definitions

```java
NPCDefinitionParser parser = new NPCDefinitionParser();
NPCDefinition npc = parser.parse("blacksmith_01", configMap);
```

---

## Services

### NPCSoundService

Event-driven sound playback:

```java
NPCSoundService soundService = new NPCSoundService(
    soundAccessor,
    eventAccessor,
    npcId -> soundBehaviors.get(npcId),
    npcId -> locations.get(npcId)
);

soundService.start();  // Subscribe to events

// Later...
soundService.stop();   // Cleanup
```

### NPCLootService

Event-driven loot drops:

```java
NPCLootService lootService = new NPCLootService(
    lootTableAccessor,
    eventAccessor,
    npcId -> lootConfigs.get(npcId),
    npcId -> wealthLevels.get(npcId)
);

lootService.start();
```

---

## Combat Behavior

### NPCCombatBehavior

Configure combat AI including aggression, factions, and abilities:

```java
NPCCombatBehavior combat = NPCCombatBehavior.builder()
    .combatEnabled(true)
    .aggression(NPCCombatBehavior.AggressionLevel.DEFENSIVE)
    .attackRange(2.0f)
    .chaseRange(20.0f)
    .retreatAt(0.2f)  // Retreat at 20% health
    
    // Faction relationships
    .faction("gondor")
    .hostileTo("mordor", "isengard")
    .friendlyTo("rohan", "shire")
    
    // Call for help
    .callsForHelp(true)
    .helpCallRange(25.0f)
    
    // Combat style
    .combatStyle(NPCCombatBehavior.CombatStyle.MELEE)
    .ability("power_strike")
    .ability("shield_bash")
    
    // Blocking
    .canBlock(true)
    .blockChance(0.3f)
    .build();

// Presets
NPCCombatBehavior.pacifist();       // Will never fight
NPCCombatBehavior.defensive("gondor");  // Fights when attacked
NPCCombatBehavior.aggressive("mordor", "gondor", "rohan");  // Attacks on sight
NPCCombatBehavior.guard("city_guard");  // Defends area
```

### Aggression Levels

| Level | Behavior |
|-------|----------|
| `PASSIVE` | Will never attack |
| `DEFENSIVE` | Only attacks when attacked first |
| `AGGRESSIVE` | Attacks hostiles on sight |
| `BERSERK` | Attacks everything on sight |

---

## Schedule Behavior

### NPCScheduleBehavior

Define time-based activities for NPCs:

```java
NPCScheduleBehavior schedule = NPCScheduleBehavior.builder()
    .at("06:00", ScheduleAction.WAKE_UP, null)
    .at("07:00", ScheduleAction.WORK, "shop_counter")
    .at("12:00", ScheduleAction.BREAK, "tavern")
    .at("13:00", ScheduleAction.WORK, "shop_counter")
    .at("18:00", ScheduleAction.REST, "home")
    .at("21:00", ScheduleAction.SLEEP, "bedroom")
    .loopDaily(true)
    .defaultActivity("IDLE")
    .build();

// Query current activity
Optional<ScheduleEntry> activity = schedule.getCurrentActivity(LocalTime.of(10, 30));
// Returns: WORK at shop_counter

// Check if NPC is active
boolean active = schedule.isActiveAt(LocalTime.of(23, 0));
// Returns: false (sleeping)

// Presets
NPCScheduleBehavior.shopkeeper();      // Standard shop hours
NPCScheduleBehavior.nightGuard();      // Active at night
NPCScheduleBehavior.farmer();          // Early riser, works fields
NPCScheduleBehavior.travelingMerchant(); // Present only during day
```

### Schedule Actions

| Action | Description |
|--------|-------------|
| `WAKE_UP` | NPC wakes and becomes active |
| `WORK` | NPC goes to work location |
| `BREAK` | NPC takes a break |
| `REST` | Reduced interaction |
| `SLEEP` | No interaction |
| `PATROL` | Follows patrol path |
| `TRADE` | Trading activities |
| `ARRIVE` / `LEAVE` | For traveling NPCs |
| `SPAWN` / `DESPAWN` | Visibility control |

---

## Social Behavior

### NPCSocialBehavior

Configure social interactions:

```java
NPCSocialBehavior social = NPCSocialBehavior.builder()
    .awarenessRadius(10.0f)
    .interactionRadius(3.0f)
    .greetsPlayers(true)
    .greetingCooldown(60)  // seconds
    .remembersPlayers(true)
    .tracksReputation(true)
    .personality(SocialPersonality.FRIENDLY)
    
    // Reputation-gated dialogs
    .requireReputation("secret_quest", 500)
    .requireReputation("advanced_items", 200)
    .unlockDialog("basic_greeting")
    .build();

// Check dialog availability
boolean canAccess = social.isDialogUnlocked("secret_quest", playerRep);

// Get all available dialogs
Set<String> dialogs = social.getAvailableDialogs(playerReputation);

// Presets
NPCSocialBehavior.friendly();
NPCSocialBehavior.formal();
NPCSocialBehavior.shy();
NPCSocialBehavior.hostile();
```

---

## Trade Behavior

### NPCTradeBehavior

Configure merchant/vendor NPCs:

```java
NPCTradeBehavior trade = NPCTradeBehavior.builder()
    .tradingEnabled(true)
    .tradeList("blacksmith_goods")
    .restockInterval(3600)  // 1 hour
    .reputationRequired(100)
    
    // Dynamic pricing
    .dynamicPricing(true)
    .minPriceMultiplier(0.5f)
    .maxPriceMultiplier(2.0f)
    
    // Haggling
    .hagglingEnabled(true)
    .maxDiscount(0.15f)  // 15% max discount
    .maxHaggleAttempts(3)
    
    // Currencies
    .acceptCurrency("gold")
    .acceptCurrency("silver")
    
    // Faction discounts
    .factionDiscount("gondor", 0.1f)  // 10% off
    .factionDiscount("shire", 0.2f)   // 20% off
    .build();

// Price calculation
int finalPrice = trade.calculatePrice(basePrice, supplyModifier, playerFaction);

// Presets
NPCTradeBehavior.disabled();
NPCTradeBehavior.shopkeeper("general_store");
NPCTradeBehavior.travelingMerchant("rare_goods");
NPCTradeBehavior.blackMarket("contraband");
```

---

## NPC Instance Persistence

### NPCInstanceData

For persisting dynamic NPC state across server restarts:

```java
NPCInstanceData instance = new NPCInstanceData();
instance.setNpcInstanceId(UUID.randomUUID());
instance.setNpcTemplateId("blacksmith_thorin");
instance.setCustomName("Thorin the Wise");

// Location
instance.setLocationX(100.0);
instance.setLocationY(64.0);
instance.setLocationZ(200.0);
instance.setWorldName("overworld");

// State
instance.setCurrentBehavior("WORKING");
instance.setAlive(true);
instance.setHealth(100.0);

// Inventory (for merchants)
instance.getInventory().put("iron_sword", 5);
instance.setCurrencyAmount(500L);

// Quest state
instance.setQuestGiver(true);
instance.getAvailableQuests().put("forge_the_ring", true);

// Interaction tracking
instance.getPlayerInteractionCounts().put(playerId, 10);
```

---

## Best Practices

1. **Use Builders**: Always use the provided builders for complex objects
2. **Configure via YAML**: Load NPC definitions from config files for easy editing
3. **Register Services**: Start services during plugin initialization
4. **Clean Shutdown**: Call `stop()` on services during plugin shutdown
5. **Use Presets**: Leverage presets like `NPCMovementBehavior.walking()` for common patterns
6. **Separate Concerns**: Define NPCs in data files, not hardcoded in Java

---

## Related Documentation

- [NPC Narrative Generator](./npc-narrative-generator.md)
- [Quest Framework](./quest-framework.md)
- [Accessor Pattern](../architecture/accessor-pattern.md)
- [Configuration Management](./config-framework.md)
