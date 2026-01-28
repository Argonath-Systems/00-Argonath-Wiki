# NPC Framework Tutorials

This section provides step-by-step tutorials for common NPC implementation scenarios.

## Table of Contents

1. [Creating Your First NPC](#creating-your-first-npc)
2. [Shopkeeper with Trading](#shopkeeper-with-trading)
3. [Patrol Guard with Combat](#patrol-guard-with-combat)
4. [Scheduled NPC (Day/Night Cycle)](#scheduled-npc-daynight-cycle)
5. [Quest Giver NPC](#quest-giver-npc)
6. [Loading NPCs from Configuration](#loading-npcs-from-configuration)

---

## Creating Your First NPC

This tutorial shows how to create a simple NPC using the NPC Framework.

### Step 1: Define the NPC

```java
import com.argonathsystems.framework.npc.api.NPCDefinitionBuilder;
import com.argonathsystems.framework.npc.spawn.*;
import com.argonathsystems.framework.npc.behavior.*;

public class MyFirstNPC {
    
    public NPCDefinition createVillager() {
        return NPCDefinitionBuilder.create("villager_tom")
            .displayName("Tom the Farmer")
            .profession("farmer")
            .wealthLevel(1)
            .backstory("A humble farmer who has worked these lands for generations.")
            
            // Set appearance
            .appearance(NPCAppearance.builder()
                .entityType("hytale:human")
                .skin("skins/farmer_male")
                .mainHand("item:hoe")
                .build())
            
            // Set spawn location
            .spawn(NPCSpawnConfig.at("overworld", 100, 64, 200))
            
            // Enable interaction
            .interactable(true)
            .invulnerable(true)
            .build();
    }
}
```

### Step 2: Register with NPC Manager

```java
// In your plugin initialization
NPCManager npcManager = new NPCManager(entityAccessor, eventAccessor);
npcManager.register(createVillager());
npcManager.spawn("villager_tom");
```

---

## Shopkeeper with Trading

Create a merchant NPC that sells items with dynamic pricing.

### Step 1: Define Trade Behavior

```java
NPCTradeBehavior tradeBehavior = NPCTradeBehavior.builder()
    .tradingEnabled(true)
    .tradeList("trades/blacksmith_goods")
    .restockInterval(3600)  // Restock every hour
    .dynamicPricing(true)
    .minPriceMultiplier(0.8f)
    .maxPriceMultiplier(1.5f)
    .hagglingEnabled(true)
    .maxDiscount(0.15f)
    .factionDiscount("guild_merchant", 0.1f)
    .build();
```

### Step 2: Define Social Behavior

```java
NPCSocialBehavior socialBehavior = NPCSocialBehavior.builder()
    .greetsPlayers(true)
    .greetingCooldown(120)
    .awarenessRadius(8.0f)
    .personality(NPCSocialBehavior.SocialPersonality.FRIENDLY)
    .tracksReputation(true)
    .requireReputation("rare_items", 500)
    .build();
```

### Step 3: Complete Definition

```java
NPCDefinition shopkeeper = NPCDefinitionBuilder.create("blacksmith_thorin")
    .displayName("Thorin Ironforge")
    .profession("blacksmith")
    .wealthLevel(3)
    
    .appearance(NPCAppearance.builder()
        .entityType("hytale:dwarf")
        .skin("skins/dwarf_blacksmith")
        .mainHand("item:blacksmith_hammer")
        .build())
    
    .soundBehavior(NPCSoundBehavior.builder()
        .onInteract("npc:dwarf_greet")
        .addAmbient("working", "sfx:hammering", 0.5f, 10, 30)
        .build())
    
    .spawn(NPCSpawnConfig.builder()
        .world("overworld")
        .position(150, 64, 220)
        .spawnType(SpawnType.PERSISTENT)
        .build())
    
    .build();
```

---

## Patrol Guard with Combat

Create a guard that patrols an area and engages hostiles.

### Step 1: Define Patrol Path

```java
PatrolPath guardRoute = PatrolPath.builder("town_square_patrol")
    .waypoint(PatrolWaypoint.named("Gate", new LocationData("world", 100, 64, 100)))
    .waypoint(PatrolWaypoint.withPause(new LocationData("world", 120, 64, 100), 10))
    .waypoint(PatrolWaypoint.named("Market", new LocationData("world", 120, 64, 120)))
    .waypoint(PatrolWaypoint.withAction(new LocationData("world", 100, 64, 120), "look_around"))
    .mode(PatrolPath.PatrolMode.CIRCULAR)
    .pauseAtWaypoints(true)
    .defaultPauseSeconds(5)
    .build();
```

### Step 2: Define Combat Behavior

```java
NPCCombatBehavior combatBehavior = NPCCombatBehavior.builder()
    .combatEnabled(true)
    .aggression(NPCCombatBehavior.AggressionLevel.DEFENSIVE)
    .faction("town_guard")
    .hostileTo("bandits", "orcs", "goblins")
    .friendlyTo("merchants", "civilians")
    .attackRange(2.5f)
    .chaseRange(25.0f)
    .callsForHelp(true)
    .helpCallRange(30.0f)
    .combatStyle(NPCCombatBehavior.CombatStyle.MELEE)
    .canBlock(true)
    .blockChance(0.3f)
    .retreatAt(0.2f)  // Retreat at 20% health
    .build();
```

### Step 3: Start Services

```java
// Initialize services
PatrolService patrolService = new PatrolService(
    entityAccessor,
    npcId -> patrolPaths.get(npcId),
    npcId -> entityIds.get(npcId),
    npcId -> locations.get(npcId),
    (npcId, action) -> handleWaypointAction(npcId, action)
);

NPCCombatService combatService = new NPCCombatService(
    entityAccessor,
    eventAccessor,
    npcId -> combatBehaviors.get(npcId),
    npcId -> entityIds.get(npcId),
    npcId -> locations.get(npcId),
    npcId -> healthPercents.get(npcId)
);

// Start with 200ms tick intervals
patrolService.start(200);
combatService.start(100);

// Register the guard
patrolService.startPatrol("guard_marcus");
combatService.registerNPC("guard_marcus");
```

---

## Scheduled NPC (Day/Night Cycle)

Create an NPC that follows a daily schedule.

### Step 1: Define Schedule

```java
NPCScheduleBehavior schedule = NPCScheduleBehavior.builder()
    // Morning routine
    .at("06:00", ScheduleAction.WAKE_UP, null)
    .at("06:30", ScheduleAction.EAT, "tavern_table")
    
    // Work day
    .at("07:00", ScheduleAction.WORK, "shop_counter")
    .at("12:00", ScheduleAction.BREAK, "tavern_table")
    .at("13:00", ScheduleAction.WORK, "shop_counter")
    
    // Evening
    .at("18:00", ScheduleAction.REST, "home_living_room")
    .at("19:00", ScheduleAction.SOCIALIZE, "tavern_bar")
    
    // Night
    .at("22:00", ScheduleAction.SLEEP, "home_bedroom")
    
    .loopDaily(true)
    .build();
```

### Step 2: Configure Schedule Service

```java
NPCScheduleService scheduleService = new NPCScheduleService(
    entityAccessor,
    eventAccessor,
    npcId -> schedules.get(npcId),
    npcId -> entityIds.get(npcId),
    locationId -> locationRegistry.get(locationId),
    () -> gameTimeProvider.getCurrentTime()
);

scheduleService.start(1000);  // Check every second
scheduleService.registerNPC("shopkeeper_marcus");
```

### Step 3: Location Registry

```java
// Register named locations
locationRegistry.put("shop_counter", new LocationData("world", 150, 64, 220));
locationRegistry.put("tavern_table", new LocationData("world", 180, 64, 200));
locationRegistry.put("home_bedroom", new LocationData("world", 140, 68, 250));
// ... etc
```

---

## Quest Giver NPC

Create an NPC that gives and tracks quests.

### Step 1: Define with Dialog

```java
NPCDefinition questGiver = NPCDefinitionBuilder.create("elder_moira")
    .displayName("Elder Moira")
    .profession("village_elder")
    .wealthLevel(2)
    .backstory("The wise elder who has guided this village for decades.")
    
    // Configure for quest interaction
    .dialogTreePath("dialogs/elder_moira/main")
    .interactable(true)
    .invulnerable(true)
    
    // Social behavior with reputation
    .build();

NPCSocialBehavior questGiverSocial = NPCSocialBehavior.builder()
    .greetsPlayers(true)
    .personality(NPCSocialBehavior.SocialPersonality.FORMAL)
    .tracksReputation(true)
    .remembersPlayers(true)
    
    // Unlock dialogs based on reputation
    .unlockDialog("greeting")
    .requireReputation("side_quest_1", 100)
    .requireReputation("main_quest_1", 250)
    .requireReputation("secret_quest", 1000)
    .build();
```

### Step 2: Handle Interaction Events

```java
eventAccessor.register(NPCInteractEvent.class, event -> {
    if (event.npcId().equals("elder_moira") && 
        event.interactionType() == InteractionType.CLICK) {
        
        UUID playerId = event.playerId();
        int reputation = reputationService.getReputation(playerId, "village");
        
        Set<String> availableDialogs = questGiverSocial.getAvailableDialogs(reputation);
        
        // Open dialog UI with available options
        dialogService.openDialog(playerId, "elder_moira", availableDialogs);
    }
});
```

---

## Loading NPCs from Configuration

Load NPCs from YAML files for easy content creation.

### Step 1: YAML Configuration

**npcs/blacksmith.yml**:
```yaml
id: blacksmith_thorin
display_name: "Thorin Ironforge"
profession: blacksmith
wealth_level: 3

origin:
  geographic: city
  cultural: dwarven
  social: artisan

appearance:
  entity_type: hytale:dwarf
  skin: skins/dwarf_blacksmith
  equipment:
    main_hand: item:blacksmith_hammer

behaviors:
  sound:
    on_interact: npc:dwarf_greet
    on_death: npc:dwarf_death
  movement:
    style: STATIONARY
  trade:
    enabled: true
    trade_list: trades/blacksmith_goods
    restock_interval: 3600

spawn:
  world: overworld
  x: 150
  y: 64
  z: 220
  type: PERSISTENT
```

### Step 2: Loading Code

```java
// Load all NPCs from directory
Path npcsDir = Paths.get("config/npcs");
NPCConfigLoader configLoader = new NPCConfigLoader(configManager);
NPCDefinitionParser parser = new NPCDefinitionParser();

Files.walk(npcsDir)
    .filter(p -> p.toString().endsWith(".yml"))
    .forEach(path -> {
        Map<String, Object> config = configManager.loadYaml(path);
        String npcId = (String) config.get("id");
        
        NPCDefinition npc = parser.parse(npcId, config);
        npcManager.register(npc);
        
        logger.info("Loaded NPC: " + npc.getDisplayName());
    });
```

### Step 3: Hot Reloading

```java
// Watch for config changes
configWatcher.watch(npcsDir, event -> {
    Path changed = event.getPath();
    String npcId = extractNpcId(changed);
    
    // Reload the NPC
    Map<String, Object> config = configManager.loadYaml(changed);
    NPCDefinition updated = parser.parse(npcId, config);
    
    npcManager.update(npcId, updated);
    logger.info("Reloaded NPC: " + npcId);
});
```

---

## Next Steps

- [NPC Framework API Reference](./npc-framework.md)
- [Configuration Reference](./npc-configuration.md)
- [Event System Guide](./event-system.md)
- [Creating Custom Behaviors](./custom-behaviors.md)
