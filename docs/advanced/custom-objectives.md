# Custom Objectives

Advanced guide to creating custom objective types in the Argonath Systems framework.

## Table of Contents

- [Overview](#overview)
- [Objective Architecture](#objective-architecture)
- [Creating Custom Objectives](#creating-custom-objectives)
- [Progress Tracking](#progress-tracking)
- [Objective Builders](#objective-builders)
- [Registration and Discovery](#registration-and-discovery)
- [Example Implementations](#example-implementations)
- [Testing and Debugging](#testing-and-debugging)
- [Best Practices](#best-practices)

## Overview

Custom objectives allow you to define unique quest goals specific to your mod's gameplay mechanics. The Argonath framework provides a flexible objective system that supports progress tracking, serialization, and event-driven updates.

### When to Create Custom Objectives

- **Unique Gameplay**: Mechanics not covered by built-in objectives
- **Complex Tracking**: Multi-stage or composite goals
- **Performance Optimization**: Specialized tracking for high-frequency events
- **Integration**: Connecting with other mods or systems

## Objective Architecture

### Core Interfaces

```java
package com.argonath.framework.objective.api;

/**
 * Core interface for quest objectives.
 * Objectives track player progress towards a specific goal.
 */
public interface Objective {
    
    /**
     * Returns the unique identifier for this objective instance.
     */
    String getId();
    
    /**
     * Returns the type identifier for this objective class.
     */
    String getType();
    
    /**
     * Returns current progress for the given player.
     */
    ObjectiveProgress getProgress(UUID playerId);
    
    /**
     * Updates progress based on an event or action.
     * 
     * @param playerId The player whose progress to update
     * @param event The triggering event
     * @return true if progress changed, false otherwise
     */
    boolean updateProgress(UUID playerId, ObjectiveEvent event);
    
    /**
     * Checks if the objective is complete for the given player.
     */
    boolean isComplete(UUID playerId);
    
    /**
     * Resets objective progress for the given player.
     */
    void reset(UUID playerId);
    
    /**
     * Returns a human-readable description.
     */
    String getDescription();
    
    /**
     * Serializes this objective to configuration format.
     */
    ConfigNode serialize();
}
```

### Progress Tracking

```java
package com.argonath.framework.objective.api;

/**
 * Represents progress towards an objective.
 * Immutable snapshot of current state.
 */
public interface ObjectiveProgress {
    
    /**
     * Returns the current progress value (e.g., items collected).
     */
    double getCurrent();
    
    /**
     * Returns the target value needed for completion.
     */
    double getTarget();
    
    /**
     * Returns progress as a percentage (0.0 to 1.0).
     */
    default double getPercentage() {
        return getTarget() > 0 ? getCurrent() / getTarget() : 0.0;
    }
    
    /**
     * Returns true if current >= target.
     */
    default boolean isComplete() {
        return getCurrent() >= getTarget();
    }
    
    /**
     * Returns optional metadata about progress.
     */
    Map<String, Object> getMetadata();
}
```

### Objective Events

```java
package com.argonath.framework.objective.api;

/**
 * Event that may trigger objective progress updates.
 */
public interface ObjectiveEvent {
    
    /**
     * Returns the event type (e.g., "item_collected", "mob_killed").
     */
    String getEventType();
    
    /**
     * Returns the player who triggered the event.
     */
    UUID getPlayerId();
    
    /**
     * Returns the timestamp when the event occurred.
     */
    long getTimestamp();
    
    /**
     * Retrieves event-specific data.
     */
    <T> Optional<T> getData(String key, Class<T> type);
}
```

## Creating Custom Objectives

### Step 1: Define the Objective Class

```java
package com.example.mymod.objectives;

import com.argonath.framework.objective.api.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Objective that requires collecting a specific quantity of items.
 */
public class CollectItemsObjective implements Objective {
    
    public static final String TYPE = "mymod:collect_items";
    
    private final String objectiveId;
    private final String itemId;
    private final int targetQuantity;
    private final boolean consumeItems;
    
    // Thread-safe progress storage
    private final ConcurrentHashMap<UUID, ItemProgress> progressMap = new ConcurrentHashMap<>();
    
    public CollectItemsObjective(String id, String itemId, int quantity, boolean consume) {
        this.objectiveId = Objects.requireNonNull(id, "id");
        this.itemId = Objects.requireNonNull(itemId, "itemId");
        this.targetQuantity = quantity;
        this.consumeItems = consume;
        
        if (quantity <= 0) {
            throw new IllegalArgumentException("Target quantity must be positive");
        }
    }
    
    @Override
    public String getId() {
        return objectiveId;
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        ItemProgress progress = progressMap.getOrDefault(playerId, new ItemProgress(0));
        return new SimpleProgress(progress.collected, targetQuantity);
    }
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        if (!"item_collected".equals(event.getEventType())) {
            return false; // Not relevant to this objective
        }
        
        // Check if the collected item matches our target
        Optional<String> collectedItemId = event.getData("item_id", String.class);
        if (!collectedItemId.map(itemId::equals).orElse(false)) {
            return false; // Wrong item type
        }
        
        int quantity = event.getData("quantity", Integer.class).orElse(1);
        
        // Update progress atomically
        ItemProgress oldProgress = progressMap.compute(playerId, (id, current) -> {
            int currentCollected = current != null ? current.collected : 0;
            int newCollected = Math.min(currentCollected + quantity, targetQuantity);
            return new ItemProgress(newCollected);
        });
        
        // Return true if progress actually changed
        return oldProgress == null || oldProgress.collected < targetQuantity;
    }
    
    @Override
    public boolean isComplete(UUID playerId) {
        return getProgress(playerId).isComplete();
    }
    
    @Override
    public void reset(UUID playerId) {
        progressMap.remove(playerId);
    }
    
    @Override
    public String getDescription() {
        String action = consumeItems ? "Consume" : "Collect";
        return String.format("%s %d × %s", action, targetQuantity, itemId);
    }
    
    @Override
    public ConfigNode serialize() {
        ConfigNode node = ConfigNode.root();
        node.node("id").set(objectiveId);
        node.node("type").set(TYPE);
        node.node("item").set(itemId);
        node.node("quantity").set(targetQuantity);
        node.node("consume").set(consumeItems);
        
        // Serialize progress data
        ConfigNode progressNode = node.node("progress");
        progressMap.forEach((playerId, progress) -> {
            progressNode.node(playerId.toString()).set(progress.collected);
        });
        
        return node;
    }
    
    /**
     * Internal progress tracking class.
     */
    private static class ItemProgress {
        final int collected;
        
        ItemProgress(int collected) {
            this.collected = collected;
        }
    }
}
```

### Step 2: Create a Deserializer

```java
package com.example.mymod.objectives;

import com.argonath.framework.objective.api.ObjectiveDeserializer;
import com.argonath.framework.config.api.ConfigNode;

/**
 * Deserializer for CollectItemsObjective.
 */
public class CollectItemsDeserializer implements ObjectiveDeserializer {
    
    @Override
    public String getType() {
        return CollectItemsObjective.TYPE;
    }
    
    @Override
    public Objective deserialize(ConfigNode node) throws DeserializationException {
        String id = node.node("id")
            .getString()
            .orElseThrow(() -> new DeserializationException("Missing 'id' field"));
            
        String itemId = node.node("item")
            .getString()
            .orElseThrow(() -> new DeserializationException("Missing 'item' field"));
            
        int quantity = node.node("quantity")
            .getInt()
            .orElseThrow(() -> new DeserializationException("Missing 'quantity' field"));
            
        boolean consume = node.node("consume")
            .getBoolean()
            .orElse(false);
            
        CollectItemsObjective objective = new CollectItemsObjective(id, itemId, quantity, consume);
        
        // Restore progress if available
        ConfigNode progressNode = node.node("progress");
        if (!progressNode.isVirtual()) {
            progressNode.childrenMap().forEach((key, value) -> {
                try {
                    UUID playerId = UUID.fromString(key.toString());
                    int collected = value.getInt().orElse(0);
                    
                    // Restore progress by simulating collection events
                    ObjectiveEvent event = new SimpleObjectiveEvent(
                        "item_collected",
                        playerId,
                        Map.of("item_id", itemId, "quantity", collected)
                    );
                    objective.updateProgress(playerId, event);
                } catch (IllegalArgumentException e) {
                    // Skip invalid UUID
                }
            });
        }
        
        return objective;
    }
    
    @Override
    public void validate(ConfigNode node) throws ValidationException {
        // Validate item ID exists in registry
        String itemId = node.node("item").getString()
            .orElseThrow(() -> new ValidationException("Item ID is required"));
            
        if (!ItemRegistry.isValidId(itemId)) {
            throw new ValidationException("Unknown item: " + itemId);
        }
        
        // Validate quantity is positive
        int quantity = node.node("quantity").getInt().orElse(0);
        if (quantity <= 0) {
            throw new ValidationException("Quantity must be positive, got: " + quantity);
        }
    }
}
```

## Progress Tracking

### Advanced Progress Implementation

```java
package com.example.mymod.objectives;

/**
 * Objective with multi-stage progress tracking.
 */
public class CraftingObjective implements Objective {
    
    public static final String TYPE = "mymod:crafting";
    
    private final String id;
    private final Map<String, Integer> requiredItems; // item ID -> quantity
    private final String resultItem;
    private final int targetCrafts;
    
    private final ConcurrentHashMap<UUID, CraftingProgress> progressMap = new ConcurrentHashMap<>();
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        if (!"item_crafted".equals(event.getEventType())) {
            return false;
        }
        
        // Verify crafted item matches target
        Optional<String> craftedItem = event.getData("item_id", String.class);
        if (!craftedItem.map(resultItem::equals).orElse(false)) {
            return false;
        }
        
        // Verify recipe matches required items
        Map<String, Integer> usedItems = event.getData("ingredients", Map.class).orElse(Map.of());
        if (!validateRecipe(usedItems)) {
            return false;
        }
        
        // Update progress
        CraftingProgress newProgress = progressMap.compute(playerId, (id, current) -> {
            int crafts = current != null ? current.craftCount : 0;
            return new CraftingProgress(
                Math.min(crafts + 1, targetCrafts),
                System.currentTimeMillis()
            );
        });
        
        return newProgress.craftCount <= targetCrafts;
    }
    
    private boolean validateRecipe(Map<String, Integer> usedItems) {
        // Ensure all required items are present in correct quantities
        for (Map.Entry<String, Integer> entry : requiredItems.entrySet()) {
            int required = entry.getValue();
            int used = usedItems.getOrDefault(entry.getKey(), 0);
            if (used < required) {
                return false;
            }
        }
        return true;
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        CraftingProgress progress = progressMap.getOrDefault(
            playerId, 
            new CraftingProgress(0, 0)
        );
        
        return new DetailedProgress(
            progress.craftCount,
            targetCrafts,
            Map.of(
                "last_craft_time", progress.lastCraftTime,
                "recipe", requiredItems,
                "result", resultItem
            )
        );
    }
    
    private static class CraftingProgress {
        final int craftCount;
        final long lastCraftTime;
        
        CraftingProgress(int count, long time) {
            this.craftCount = count;
            this.lastCraftTime = time;
        }
    }
}
```

### Persistent Progress Storage

```java
package com.example.mymod.objectives;

import com.argonath.framework.storage.api.StorageAdapter;

/**
 * Objective that persists progress to database.
 */
public abstract class PersistentObjective implements Objective {
    
    private final StorageAdapter storage;
    private final String tableName;
    
    protected PersistentObjective(StorageAdapter storage, String tableName) {
        this.storage = storage;
        this.tableName = tableName;
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        return storage.query(
            "SELECT current, target FROM " + tableName + " WHERE player_id = ? AND objective_id = ?",
            playerId.toString(),
            getId()
        ).map(result -> new SimpleProgress(
            result.getDouble("current"),
            result.getDouble("target")
        )).orElse(new SimpleProgress(0, getTargetValue()));
    }
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        double newProgress = calculateProgress(playerId, event);
        
        storage.execute(
            "INSERT INTO " + tableName + " (player_id, objective_id, current, target) " +
            "VALUES (?, ?, ?, ?) " +
            "ON CONFLICT (player_id, objective_id) DO UPDATE SET current = ?",
            playerId.toString(),
            getId(),
            newProgress,
            getTargetValue(),
            newProgress
        );
        
        return true;
    }
    
    /**
     * Subclasses implement to calculate new progress value.
     */
    protected abstract double calculateProgress(UUID playerId, ObjectiveEvent event);
    
    /**
     * Returns the target value for completion.
     */
    protected abstract double getTargetValue();
}
```

## Objective Builders

### Fluent Builder Pattern

```java
package com.example.mymod.objectives.builder;

/**
 * Builder for creating complex objectives with fluent API.
 */
public class ObjectiveBuilder {
    
    private String id;
    private String type;
    private final Map<String, Object> parameters = new HashMap<>();
    private final List<Condition> requirements = new ArrayList<>();
    private final List<String> rewards = new ArrayList<>();
    
    public static ObjectiveBuilder create(String type) {
        ObjectiveBuilder builder = new ObjectiveBuilder();
        builder.type = type;
        return builder;
    }
    
    public ObjectiveBuilder withId(String id) {
        this.id = id;
        return this;
    }
    
    public ObjectiveBuilder parameter(String key, Object value) {
        parameters.put(key, value);
        return this;
    }
    
    public ObjectiveBuilder requireCondition(Condition condition) {
        requirements.add(condition);
        return this;
    }
    
    public ObjectiveBuilder grantReward(String rewardId) {
        rewards.add(rewardId);
        return this;
    }
    
    public Objective build() {
        if (id == null) {
            id = UUID.randomUUID().toString();
        }
        
        ObjectiveFactory factory = ObjectiveRegistry.getFactory(type);
        if (factory == null) {
            throw new IllegalStateException("Unknown objective type: " + type);
        }
        
        Objective base = factory.create(id, parameters);
        
        // Wrap with requirements if present
        if (!requirements.isEmpty()) {
            base = new ConditionalObjective(base, new AndCondition(requirements));
        }
        
        // Attach rewards if present
        if (!rewards.isEmpty()) {
            base = new RewardingObjective(base, rewards);
        }
        
        return base;
    }
}

// Usage example:
Objective objective = ObjectiveBuilder.create("mymod:collect_items")
    .withId("collect_iron")
    .parameter("item", "minecraft:iron_ingot")
    .parameter("quantity", 64)
    .parameter("consume", false)
    .requireCondition(new LevelCondition(10))
    .grantReward("bonus_xp")
    .build();
```

### Type-Safe Builders

```java
package com.example.mymod.objectives.builder;

/**
 * Type-safe builder for collect items objectives.
 */
public class CollectItemsBuilder {
    
    private String id;
    private String itemId;
    private int quantity = 1;
    private boolean consume = false;
    
    public CollectItemsBuilder() {
        this.id = UUID.randomUUID().toString();
    }
    
    public CollectItemsBuilder id(String id) {
        this.id = Objects.requireNonNull(id);
        return this;
    }
    
    public CollectItemsBuilder item(String itemId) {
        this.itemId = Objects.requireNonNull(itemId);
        return this;
    }
    
    public CollectItemsBuilder quantity(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.quantity = quantity;
        return this;
    }
    
    public CollectItemsBuilder consume() {
        this.consume = true;
        return this;
    }
    
    public CollectItemsObjective build() {
        if (itemId == null) {
            throw new IllegalStateException("Item ID must be specified");
        }
        return new CollectItemsObjective(id, itemId, quantity, consume);
    }
}

// Usage:
CollectItemsObjective objective = new CollectItemsBuilder()
    .item("minecraft:diamond")
    .quantity(10)
    .consume()
    .build();
```

## Registration and Discovery

### Registry Pattern

```java
package com.example.mymod;

import com.argonath.framework.objective.api.ObjectiveRegistry;
import com.argonath.platform.sdk.ModInitializer;

public class MyMod implements ModInitializer {
    
    @Override
    public void onInitialize() {
        ObjectiveRegistry registry = ObjectiveRegistry.getInstance();
        
        // Register objective deserializers
        registry.register(new CollectItemsDeserializer());
        registry.register(new CraftingDeserializer());
        registry.register(new MiningDeserializer());
        registry.register(new TradingDeserializer());
        registry.register(new ExplorationDeserializer());
        
        // Register objective factories for programmatic creation
        registry.registerFactory("mymod:collect_items", CollectItemsObjective::fromParameters);
        registry.registerFactory("mymod:crafting", CraftingObjective::fromParameters);
        
        getLogger().info("Registered {} custom objective types", 5);
    }
}
```

### Event Listener Registration

```java
package com.example.mymod.listeners;

import com.argonath.platform.core.event.EventBus;
import com.argonath.platform.core.event.item.ItemCollectedEvent;
import com.argonath.platform.core.event.item.ItemCraftedEvent;

/**
 * Registers event listeners for objective updates.
 */
public class ObjectiveEventListeners {
    
    private final QuestManager questManager;
    
    public void register(EventBus eventBus) {
        // Item collection events
        eventBus.subscribe(ItemCollectedEvent.class, this::onItemCollected);
        
        // Crafting events
        eventBus.subscribe(ItemCraftedEvent.class, this::onItemCrafted);
        
        // Mining events
        eventBus.subscribe(BlockMinedEvent.class, this::onBlockMined);
        
        // Trading events
        eventBus.subscribe(TradeCompletedEvent.class, this::onTradeCompleted);
    }
    
    private void onItemCollected(ItemCollectedEvent event) {
        ObjectiveEvent objEvent = new SimpleObjectiveEvent(
            "item_collected",
            event.getPlayer().getUUID(),
            Map.of(
                "item_id", event.getItem().getId(),
                "quantity", event.getQuantity()
            )
        );
        
        questManager.notifyObjectiveEvent(event.getPlayer().getUUID(), objEvent);
    }
    
    private void onItemCrafted(ItemCraftedEvent event) {
        ObjectiveEvent objEvent = new SimpleObjectiveEvent(
            "item_crafted",
            event.getPlayer().getUUID(),
            Map.of(
                "item_id", event.getResult().getId(),
                "ingredients", event.getIngredients()
            )
        );
        
        questManager.notifyObjectiveEvent(event.getPlayer().getUUID(), objEvent);
    }
}
```

## Example Implementations

### Mining Objective

```java
package com.example.mymod.objectives;

/**
 * Objective that requires mining specific blocks.
 */
public class MiningObjective implements Objective {
    
    public static final String TYPE = "mymod:mining";
    
    private final String id;
    private final String blockType;
    private final int targetCount;
    private final boolean requireTool;
    private final String toolType; // e.g., "minecraft:diamond_pickaxe"
    
    private final ConcurrentHashMap<UUID, Integer> minedCounts = new ConcurrentHashMap<>();
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        if (!"block_mined".equals(event.getEventType())) {
            return false;
        }
        
        // Verify block type
        Optional<String> minedBlock = event.getData("block_type", String.class);
        if (!minedBlock.map(blockType::equals).orElse(false)) {
            return false;
        }
        
        // Verify tool if required
        if (requireTool) {
            Optional<String> usedTool = event.getData("tool", String.class);
            if (!usedTool.map(toolType::equals).orElse(false)) {
                return false; // Wrong tool used
            }
        }
        
        // Increment counter
        int newCount = minedCounts.compute(playerId, (id, current) -> {
            int count = current != null ? current : 0;
            return Math.min(count + 1, targetCount);
        });
        
        return newCount <= targetCount;
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        int current = minedCounts.getOrDefault(playerId, 0);
        return new SimpleProgress(current, targetCount);
    }
    
    @Override
    public String getDescription() {
        String toolReq = requireTool ? " with " + toolType : "";
        return String.format("Mine %d × %s%s", targetCount, blockType, toolReq);
    }
}
```

### Trading Objective

```java
package com.example.mymod.objectives;

/**
 * Objective that requires trading with NPCs or players.
 */
public class TradingObjective implements Objective {
    
    public static final String TYPE = "mymod:trading";
    
    private final String id;
    private final String npcId; // null for any NPC
    private final List<TradeRequirement> trades;
    
    private final ConcurrentHashMap<UUID, Set<String>> completedTrades = new ConcurrentHashMap<>();
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        if (!"trade_completed".equals(event.getEventType())) {
            return false;
        }
        
        // Verify NPC if specified
        if (npcId != null) {
            Optional<String> tradedWith = event.getData("npc_id", String.class);
            if (!tradedWith.map(npcId::equals).orElse(false)) {
                return false;
            }
        }
        
        // Get trade details
        String tradeId = event.getData("trade_id", String.class).orElse(null);
        if (tradeId == null) {
            return false;
        }
        
        // Check if this trade is required
        boolean isRequired = trades.stream()
            .anyMatch(req -> req.tradeId.equals(tradeId));
            
        if (!isRequired) {
            return false;
        }
        
        // Mark trade as completed
        completedTrades.computeIfAbsent(playerId, k -> ConcurrentHashMap.newKeySet())
            .add(tradeId);
            
        return true;
    }
    
    @Override
    public boolean isComplete(UUID playerId) {
        Set<String> completed = completedTrades.getOrDefault(playerId, Set.of());
        return trades.stream()
            .allMatch(req -> completed.contains(req.tradeId));
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        Set<String> completed = completedTrades.getOrDefault(playerId, Set.of());
        int current = (int) trades.stream()
            .filter(req -> completed.contains(req.tradeId))
            .count();
        return new SimpleProgress(current, trades.size());
    }
    
    public static class TradeRequirement {
        final String tradeId;
        final String itemGiven;
        final int quantityGiven;
        final String itemReceived;
        final int quantityReceived;
        
        // Constructor and methods...
    }
}
```

### Exploration Objective

```java
package com.example.mymod.objectives;

/**
 * Objective that requires discovering specific locations.
 */
public class ExplorationObjective implements Objective {
    
    public static final String TYPE = "mymod:exploration";
    
    private final String id;
    private final List<Location> targetLocations;
    private final double discoveryRadius; // How close player must get
    
    private final ConcurrentHashMap<UUID, Set<Location>> discoveredLocations = 
        new ConcurrentHashMap<>();
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        if (!"player_moved".equals(event.getEventType())) {
            return false;
        }
        
        Optional<Location> playerLoc = event.getData("location", Location.class);
        if (!playerLoc.isPresent()) {
            return false;
        }
        
        Location loc = playerLoc.get();
        Set<Location> discovered = discoveredLocations.computeIfAbsent(
            playerId, 
            k -> ConcurrentHashMap.newKeySet()
        );
        
        boolean updated = false;
        for (Location target : targetLocations) {
            if (!discovered.contains(target) && isWithinRadius(loc, target, discoveryRadius)) {
                discovered.add(target);
                updated = true;
                
                // Send discovery notification
                notifyLocationDiscovered(playerId, target);
            }
        }
        
        return updated;
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        int discovered = discoveredLocations.getOrDefault(playerId, Set.of()).size();
        return new DetailedProgress(
            discovered,
            targetLocations.size(),
            Map.of("locations", targetLocations, "radius", discoveryRadius)
        );
    }
    
    private boolean isWithinRadius(Location a, Location b, double radius) {
        return a.distanceTo(b) <= radius;
    }
    
    private void notifyLocationDiscovered(UUID playerId, Location location) {
        // Send notification to player
        MessageService.getInstance().sendMessage(
            playerId,
            "Discovery!",
            "You have discovered: " + location.getName()
        );
    }
}
```

## Testing and Debugging

### Unit Testing

```java
package com.example.mymod.objectives;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CollectItemsObjectiveTest {
    
    private CollectItemsObjective objective;
    private UUID testPlayerId;
    
    @BeforeEach
    void setUp() {
        objective = new CollectItemsObjective(
            "test_collect",
            "minecraft:diamond",
            10,
            false
        );
        testPlayerId = UUID.randomUUID();
    }
    
    @Test
    @DisplayName("Should start with zero progress")
    void testInitialProgress() {
        ObjectiveProgress progress = objective.getProgress(testPlayerId);
        assertEquals(0, progress.getCurrent());
        assertEquals(10, progress.getTarget());
        assertFalse(progress.isComplete());
    }
    
    @Test
    @DisplayName("Should update progress on item collection")
    void testProgressUpdate() {
        ObjectiveEvent event = createCollectionEvent("minecraft:diamond", 5);
        
        boolean updated = objective.updateProgress(testPlayerId, event);
        
        assertTrue(updated);
        assertEquals(5, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @Test
    @DisplayName("Should not exceed target quantity")
    void testMaxProgress() {
        ObjectiveEvent event1 = createCollectionEvent("minecraft:diamond", 8);
        ObjectiveEvent event2 = createCollectionEvent("minecraft:diamond", 5);
        
        objective.updateProgress(testPlayerId, event1);
        objective.updateProgress(testPlayerId, event2);
        
        assertEquals(10, objective.getProgress(testPlayerId).getCurrent());
        assertTrue(objective.isComplete(testPlayerId));
    }
    
    @Test
    @DisplayName("Should ignore wrong item type")
    void testWrongItemType() {
        ObjectiveEvent event = createCollectionEvent("minecraft:iron_ingot", 10);
        
        boolean updated = objective.updateProgress(testPlayerId, event);
        
        assertFalse(updated);
        assertEquals(0, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @Test
    @DisplayName("Should reset progress correctly")
    void testReset() {
        ObjectiveEvent event = createCollectionEvent("minecraft:diamond", 5);
        objective.updateProgress(testPlayerId, event);
        
        objective.reset(testPlayerId);
        
        assertEquals(0, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @Test
    @DisplayName("Should serialize and deserialize correctly")
    void testSerialization() throws Exception {
        // Add some progress
        ObjectiveEvent event = createCollectionEvent("minecraft:diamond", 7);
        objective.updateProgress(testPlayerId, event);
        
        // Serialize
        ConfigNode serialized = objective.serialize();
        
        // Deserialize
        CollectItemsObjective deserialized = 
            (CollectItemsObjective) new CollectItemsDeserializer().deserialize(serialized);
        
        // Verify
        assertEquals(objective.getId(), deserialized.getId());
        assertEquals(7, deserialized.getProgress(testPlayerId).getCurrent());
    }
    
    private ObjectiveEvent createCollectionEvent(String itemId, int quantity) {
        return new SimpleObjectiveEvent(
            "item_collected",
            testPlayerId,
            Map.of("item_id", itemId, "quantity", quantity)
        );
    }
}
```

### Integration Testing

```java
package com.example.mymod.objectives.integration;

import com.argonath.framework.test.IntegrationTest;
import com.argonath.framework.test.QuestTestEnvironment;

@IntegrationTest
class ObjectiveIntegrationTest {
    
    private QuestTestEnvironment env;
    
    @BeforeEach
    void setUp() {
        env = QuestTestEnvironment.create();
    }
    
    @Test
    @DisplayName("Full objective lifecycle test")
    void testObjectiveLifecycle() {
        // Create test player
        TestPlayer player = env.createPlayer("TestPlayer");
        
        // Create quest with objective
        Quest quest = new QuestBuilder()
            .id("test_quest")
            .objective(new CollectItemsBuilder()
                .item("minecraft:diamond")
                .quantity(5)
                .build())
            .build();
            
        // Start quest
        env.startQuest(player, quest);
        
        // Simulate collecting items
        env.giveItem(player, "minecraft:diamond", 3);
        
        // Verify progress
        ObjectiveProgress progress = env.getObjectiveProgress(player, quest, 0);
        assertEquals(3, progress.getCurrent());
        assertFalse(progress.isComplete());
        
        // Collect remaining items
        env.giveItem(player, "minecraft:diamond", 2);
        
        // Verify completion
        assertTrue(env.isObjectiveComplete(player, quest, 0));
        assertTrue(env.isQuestComplete(player, quest));
    }
}
```

### Debugging Tools

```java
package com.example.mymod.debug;

/**
 * Debugging utilities for objectives.
 */
public class ObjectiveDebugger {
    
    /**
     * Prints detailed objective state for debugging.
     */
    public static void debugObjective(Objective objective, UUID playerId) {
        System.out.println("=== Objective Debug Info ===");
        System.out.println("ID: " + objective.getId());
        System.out.println("Type: " + objective.getType());
        System.out.println("Description: " + objective.getDescription());
        
        ObjectiveProgress progress = objective.getProgress(playerId);
        System.out.println("Progress: " + progress.getCurrent() + " / " + progress.getTarget());
        System.out.println("Percentage: " + (progress.getPercentage() * 100) + "%");
        System.out.println("Complete: " + progress.isComplete());
        
        if (!progress.getMetadata().isEmpty()) {
            System.out.println("Metadata:");
            progress.getMetadata().forEach((key, value) -> 
                System.out.println("  " + key + ": " + value)
            );
        }
        System.out.println("===========================");
    }
}
```

## Best Practices

### 1. Thread Safety

Always use concurrent data structures for progress tracking:

```java
private final ConcurrentHashMap<UUID, Progress> progressMap = new ConcurrentHashMap<>();
```

### 2. Immutable Progress

Return immutable progress snapshots:

```java
@Override
public ObjectiveProgress getProgress(UUID playerId) {
    Progress internal = progressMap.get(playerId);
    return new ImmutableProgress(internal); // Defensive copy
}
```

### 3. Event Filtering

Filter events early to avoid unnecessary processing:

```java
@Override
public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
    if (!isRelevantEvent(event)) {
        return false; // Early return
    }
    // ... rest of logic
}
```

### 4. Progress Capping

Always cap progress at target value:

```java
int newValue = Math.min(current + increment, targetValue);
```

### 5. Validation

Validate all inputs in constructors:

```java
public MyObjective(String id, int target) {
    this.id = Objects.requireNonNull(id, "ID cannot be null");
    if (target <= 0) {
        throw new IllegalArgumentException("Target must be positive");
    }
    this.target = target;
}
```

### 6. Resource Cleanup

Clean up resources when objectives complete:

```java
@Override
public void reset(UUID playerId) {
    progressMap.remove(playerId);
    // Clean up any event listeners, caches, etc.
}
```

### 7. Performance Monitoring

Track objective performance:

```java
@Override
public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
    long start = System.nanoTime();
    try {
        return doUpdateProgress(playerId, event);
    } finally {
        long duration = System.nanoTime() - start;
        metrics.recordUpdateDuration(getType(), duration);
    }
}
```

## See Also

- [Custom Conditions](custom-conditions.md) - Creating custom condition types
- [Performance Optimization](performance.md) - Advanced performance tuning
- [Testing Strategies](testing.md) - Comprehensive testing guide
- [API Reference](../api/framework-objective.md) - Objective API documentation
