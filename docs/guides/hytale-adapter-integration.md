# Hytale Adapter Integration Guide

**Version:** 1.0.0  
**Last Updated:** 2026-01-28  
**Status:** ✅ Active

---

## Overview

This guide explains how the Argonath Systems architecture integrates with Hytale, including the critical **Accessor Pattern** that enables platform-agnostic development. Understanding this integration is essential for all mod developers.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Accessor Pattern](#the-accessor-pattern)
3. [Hytale Adapter Module](#hytale-adapter-module)
4. [Event Translation](#event-translation)
5. [Data Type Conversion](#data-type-conversion)
6. [NPC Integration with Hytale](#npc-integration-with-hytale)
7. [Quest Integration with Hytale](#quest-integration-with-hytale)
8. [Dialogue System Communication](#dialogue-system-communication)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

---

## Architecture Overview

Argonath Systems uses a **layered architecture** that isolates all Hytale-specific code to a single adapter module:

```
┌───────────────────────────────────────────────────────┐
│                     YOUR MOD                          │
│     (Uses ONLY framework interfaces - no hytale.*)    │
├───────────────────────────────────────────────────────┤
│              FRAMEWORK LAYER (04-05-xx)               │
│    • framework-quest    • framework-npc               │
│    • framework-ui       • framework-objective         │
│    (Platform-agnostic business logic)                 │
├───────────────────────────────────────────────────────┤
│              ACCESSOR API (02-framework-accessor)     │
│    • EntityAccessor     • PlayerAccessor              │
│    • EventAccessor      • WorldAccessor               │
│    (Interface definitions - no implementations)       │
├───────────────────────────────────────────────────────┤
│              HYTALE ADAPTER (02-adapter-hytale)       │
│    🚨 ONLY MODULE WITH hytale.* IMPORTS 🚨            │
│    (Implements all Accessor interfaces)               │
├───────────────────────────────────────────────────────┤
│                   HYTALE ENGINE                       │
│    (Game runtime - com.hypixel.hytale.*)              │
└───────────────────────────────────────────────────────┘
```

### Why This Architecture?

1. **Alpha Stability**: Hytale's API is in Alpha and will change. Isolation protects your code.
2. **Testability**: Mock accessors for unit testing without running the game.
3. **Portability**: Theoretically support other game engines.
4. **Maintainability**: API changes only affect the adapter layer.

---

## The Accessor Pattern

### Core Concept

**Accessors** are interfaces that abstract game engine operations. Your mod depends on accessor interfaces, never on Hytale classes directly.

### Available Accessors

| Accessor | Purpose |
|----------|---------|
| `EntityAccessor` | Entity creation, modification, metadata |
| `PlayerAccessor` | Player data, inventory, stats |
| `EventAccessor` | Event subscription and dispatch |
| `WorldAccessor` | World data, block operations, spawning |
| `ItemAccessor` | Item creation and manipulation |
| `SoundAccessor` | Sound playback |
| `RegistryAccessor` | Game asset registries |
| `LootTableAccessor` | Loot table querying |

### Using Accessors

```java
// ✅ CORRECT - Use accessor interfaces
public class MyQuestManager {
    private final EntityAccessor entityAccessor;
    private final EventAccessor eventAccessor;
    
    public MyQuestManager(EntityAccessor entityAccessor, EventAccessor eventAccessor) {
        this.entityAccessor = entityAccessor;
        this.eventAccessor = eventAccessor;
    }
    
    public void spawnQuestNPC(String npcId, LocationData location) {
        entityAccessor.spawnEntity("hytale:human", location);
    }
}

// ❌ FORBIDDEN - Never import Hytale classes in frameworks/mods
import com.hypixel.hytale.entity.Entity;  // NEVER DO THIS
```

### Dependency Injection

Accessors are injected at runtime by the adapter:

```java
// In your plugin initialization
public class MyModPlugin extends JavaPluginInit {
    
    @Override
    public void onEnable(JavaPlugin plugin) {
        // Get accessor provider
        AccessorProvider provider = AccessorProvider.getInstance();
        
        // Get accessor implementations
        EntityAccessor entityAccessor = provider.getEntityAccessor();
        EventAccessor eventAccessor = provider.getEventAccessor();
        
        // Create your services
        MyQuestManager questManager = new MyQuestManager(entityAccessor, eventAccessor);
    }
}
```

---

## Hytale Adapter Module

The `02-adapter-hytale` module is the **only place** where Hytale-specific code exists.

### Module Structure

```
02-adapter-hytale/
└── src/main/java/com/argonathsystems/adapter/hytale/
    ├── HytaleAdapterPlugin.java      # Main plugin entry
    ├── HytaleAdapterProvider.java    # AccessorProvider implementation
    ├── accessor/                     # Accessor implementations
    │   ├── HytaleEntityAccessor.java
    │   ├── HytalePlayerAccessor.java
    │   ├── HytaleEventAccessor.java
    │   ├── HytaleWorldAccessor.java
    │   └── HytaleItemAccessor.java
    ├── converter/                    # Type converters
    │   ├── EntityConverter.java
    │   ├── ItemConverter.java
    │   └── LocationConverter.java
    └── integration/                  # Quest/NPC integration helpers
        └── QuestFormatConverter.java
```

### Type Conversion

The adapter converts between Hytale types and platform-agnostic DTOs:

```java
// Hytale types (ONLY in adapter)
import com.hypixel.hytale.entity.Player;
import com.hypixel.hytale.item.ItemStack;
import com.hypixel.hytale.world.Location;

// Platform-agnostic DTOs (used everywhere else)
import com.argonathsystems.framework.accessorapi.dto.PlayerData;
import com.argonathsystems.framework.accessorapi.dto.ItemData;
import com.argonathsystems.framework.accessorapi.dto.LocationData;
```

**Conversion Examples:**

```java
// In HytaleEntityAccessor (adapter layer only)
public PlayerData getPlayerData(UUID playerId) {
    Player hytalePlayer = server.getPlayer(playerId);
    
    return new PlayerData(
        hytalePlayer.getUniqueId(),
        hytalePlayer.getName(),
        hytalePlayer.getHealth(),
        hytalePlayer.getMaxHealth(),
        convertLocation(hytalePlayer.getLocation())
    );
}

private LocationData convertLocation(Location loc) {
    return new LocationData(
        loc.getWorld().getName(),
        loc.getX(),
        loc.getY(),
        loc.getZ(),
        loc.getYaw(),
        loc.getPitch()
    );
}
```

---

## Event Translation

### How Event Translation Works

1. Hytale fires a native event
2. Adapter catches and converts it
3. Framework receives platform-agnostic event
4. Your mod reacts to the event

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│ Hytale Event │────▶│ Adapter Layer   │────▶│ Accessor Event   │
│ (Native)     │     │ (Converts)      │     │ (Platform-agnostic)
└──────────────┘     └─────────────────┘     └──────────────────┘
                                                      │
                                                      ▼
                                             ┌──────────────────┐
                                             │ Your Event       │
                                             │ Handlers         │
                                             └──────────────────┘
```

### Event Translation Example

```java
// In HytaleEventAdapter (adapter layer)
public class HytaleEventAdapter {
    
    public void init(HytaleServer server, EventAccessor eventAccessor) {
        // Listen to Hytale native events
        server.getEventBus().register(PlayerInteractEntityEvent.class, nativeEvent -> {
            // Convert to platform-agnostic event
            NPCInteractEvent accessorEvent = new NPCInteractEvent(
                nativeEvent.getPlayer().getUniqueId(),
                nativeEvent.getEntity().getId().toString(),
                InteractionType.CLICK,
                System.currentTimeMillis()
            );
            
            // Dispatch to accessor event bus
            eventAccessor.dispatch(accessorEvent);
        });
    }
}
```

### Subscribing to Events in Your Mod

```java
// In your mod (framework layer)
public class NPCInteractionHandler {
    
    public NPCInteractionHandler(EventAccessor eventAccessor) {
        // Register for platform-agnostic events
        eventAccessor.register(NPCInteractEvent.class, this::onNPCInteract);
    }
    
    private void onNPCInteract(NPCInteractEvent event) {
        String npcId = event.getNpcId();
        UUID playerId = event.getPlayerId();
        
        // Handle interaction...
        dialogueService.startDialogue(playerId, npcId);
    }
}
```

---

## NPC Integration with Hytale

### NPC Spawning Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR MOD                                 │
│  NPCDefinition npc = NPCDefinitionBuilder.create("blacksmith") │
│      .displayName("Thorin")                                     │
│      .spawn(NPCSpawnConfig.at("world", 100, 64, 200))          │
│      .build();                                                  │
│  npcManager.spawn(npc);                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     NPC FRAMEWORK                               │
│  npcManager.spawn(npc) calls:                                   │
│      entityAccessor.spawnEntity(appearance.getEntityType(), loc)│
│      entityAccessor.setMetadata(entityId, "npc_template", npcId)│
│      eventAccessor.dispatch(new NPCSpawnEvent(npcId, entityId)) │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE ADAPTER                              │
│  HytaleEntityAccessor.spawnEntity():                            │
│      Entity entity = server.getWorld(world)                     │
│          .spawn(EntityType.get(entityType), x, y, z);           │
│      return entity.getId().toString();                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE ENGINE                               │
│  Native entity created in game world                            │
└─────────────────────────────────────────────────────────────────┘
```

### NPC Entity Mapping

The adapter maintains a mapping between NPC template IDs and Hytale entity IDs:

```java
// NPC Manager maintains mappings
public class NPCManager {
    private final Map<String, String> npcToEntity = new ConcurrentHashMap<>();
    private final Map<String, String> entityToNpc = new ConcurrentHashMap<>();
    
    public void spawn(NPCDefinition definition) {
        String entityId = entityAccessor.spawnEntity(
            definition.getAppearance().get().getEntityType(),
            definition.getSpawnConfig().get().toLocationData()
        );
        
        npcToEntity.put(definition.getTemplateId(), entityId);
        entityToNpc.put(entityId, definition.getTemplateId());
    }
    
    public Optional<String> getNpcIdForEntity(String entityId) {
        return Optional.ofNullable(entityToNpc.get(entityId));
    }
}
```

---

## Quest Integration with Hytale

### Quest Objective Event Flow

When a player kills an entity for a quest:

```
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE ENGINE                               │
│  Player kills entity → EntityDeathEvent fired                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE ADAPTER                              │
│  Converts EntityDeathEvent to:                                  │
│      EntityKillEvent(killerId, victimType, location)            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   OBJECTIVE FRAMEWORK                           │
│  KillObjectiveTracker listens for EntityKillEvent               │
│  Updates progress for matching quests                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    QUEST FRAMEWORK                              │
│  QuestManager notified of objective progress                    │
│  Checks if quest can complete                                   │
│  Dispatches QuestProgressEvent                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Quest Registration with Hytale

Quests are registered through the Quest Framework, which uses the adapter for:

- **Reward Distribution**: Converting reward items to Hytale ItemStacks
- **Condition Checking**: Querying player stats through PlayerAccessor
- **Progress Persistence**: Using Storage Framework (platform-agnostic)

---

## Dialogue System Communication

### Dialogue Flow with HyUI

The dialogue system integrates with Hytale through the UI Framework:

```
┌─────────────────────────────────────────────────────────────────┐
│                     NPC FRAMEWORK                               │
│  DialogueSession session = dialogueService.startDialogue(       │
│      playerId, npcId, "greeting");                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    UI FRAMEWORK                                 │
│  DialogueUI.show(player, currentNode, options)                  │
│  Creates HyUIML document for dialogue box                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE ADAPTER                              │
│  HyUIAdapter sends HyUIML to player's client                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     HYTALE CLIENT                               │
│  Renders dialogue UI with options                               │
│  Player clicks option → UI callback fired                       │
└─────────────────────────────────────────────────────────────────┘
```

### DialogueTree Integration

```java
// Create dialogue tree (platform-agnostic)
DialogueTree blacksmithDialogue = DialogueTree.builder("blacksmith_main")
    .npcId("blacksmith_001")
    .displayName("Ironforge Blacksmith")
    .startNode("greeting")
    .node(DialogueNode.builder("greeting")
        .speaker("Thordak")
        .text("Welcome to Ironforge! Need something forged?")
        .option(DialogueOption.simple("shop", "Show me your wares", "shop_intro"))
        .option(DialogueOption.builder("quest")
            .text("I heard you need help...")
            .targetNode("quest_offer")
            .condition(ctx -> !ctx.hasCompletedQuest("blacksmith_quest"))
            .build())
        .option(DialogueOption.goodbye("Farewell"))
        .build())
    .build();

// Register dialogue
dialogueRegistry.register(blacksmithDialogue);
```

### Handling Dialogue Responses

```java
eventAccessor.register(DialogueOptionSelectedEvent.class, event -> {
    DialogueSession session = sessionManager.getSession(event.getPlayerId());
    DialogueOption selected = session.getOption(event.getOptionId());
    
    // Execute option action if any
    selected.getAction().ifPresent(action -> action.execute(session.getContext()));
    
    // Navigate to next node
    if (selected.getTargetNodeId().isPresent()) {
        session.navigateTo(selected.getTargetNodeId().get());
        dialogueUI.update(session);
    } else {
        session.end();
    }
});
```

---

## Best Practices

### DO:

- ✅ **Always use accessor interfaces** - Never import hytale.* outside the adapter
- ✅ **Use DTOs for data transfer** - LocationData, PlayerData, ItemData
- ✅ **Register event handlers through EventAccessor** - Platform-agnostic events
- ✅ **Test with mock accessors** - Unit tests without running Hytale
- ✅ **Check Optional returns** - Many accessor methods return Optional

### DON'T:

- ❌ **Import Hytale classes in frameworks/mods** - Architecture violation!
- ❌ **Store Hytale objects in long-lived fields** - Use IDs/references
- ❌ **Call Hytale API directly** - Always go through accessors
- ❌ **Assume synchronous execution** - Some operations may be async
- ❌ **Ignore API deprecations** - The Hytale API is in Alpha

### Code Review Checklist

Before committing code in frameworks/mods:

- [ ] No `import hytale.*` or `import com.hypixel.hytale.*` statements
- [ ] Using accessor interfaces for all game interactions
- [ ] DTOs used for data transfer, not engine types
- [ ] Events registered through EventAccessor
- [ ] Unit tests use mock accessors

---

## Troubleshooting

### "Accessor not found" Error

**Cause**: Adapter not loaded or not initialized before your mod.

**Solution**: Ensure `hytale-adapter` is in your dependencies and loads before your mod:

```xml
<dependency>
    <groupId>com.argonathsystems.adapter</groupId>
    <artifactId>hytale-adapter</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Events Not Firing

**Cause**: Event type mismatch or registration issue.

**Debug Steps**:
1. Verify event is registered correctly
2. Check that adapter is translating the native event
3. Enable debug logging in EventAccessor

```java
eventAccessor.setDebugMode(true);  // Log all event dispatches
```

### NPC Not Spawning

**Cause**: Entity spawn failed in Hytale.

**Debug Steps**:
1. Check spawn location is valid (loaded chunk)
2. Verify entity type exists in registry
3. Check server logs for Hytale errors

```java
// Get spawn result
SpawnResult result = entityAccessor.spawnEntityWithResult(entityType, location);
if (result.isFailure()) {
    logger.error("Spawn failed: " + result.getFailureReason());
}
```

---

## Related Documentation

- [Accessor API Reference](../api-reference/accessor-api.md)
- [NPC Framework API Reference](../api-reference/npc-framework.md)
- [Quest Development Guide](./quest-development.md)
- [Architecture Overview](../architecture/overview.md)
- [Event System Guide](./event-system.md)

---

## Summary

The Hytale integration follows these key principles:

1. **Zero Hytale imports** in business logic (frameworks/mods)
2. **Accessor Pattern** abstracts all game engine interactions
3. **Single adapter module** contains all Hytale-specific code
4. **Event translation** converts native events to platform-agnostic events
5. **DTOs** replace engine types for data transfer

This architecture protects your code from Hytale API changes while enabling full game integration.
