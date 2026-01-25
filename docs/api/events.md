# Event System API Reference

**Package:** `com.argonathsystems.platform.core.event`  
**Module:** Platform Core  
**Version:** 1.0.0

## Overview

The Argonath Systems Event System provides a high-performance, type-safe event bus for inter-module communication. Features include priority-based execution, cancellation support, asynchronous event handling, and comprehensive built-in events for quests, NPCs, players, and world interactions.

## Table of Contents

- [Event Base Class](#event-base-class)
- [EventBus](#eventbus)
- [EventListener](#eventlistener)
- [EventPriority](#eventpriority)
- [Quest Events](#quest-events)
- [NPC Events](#npc-events)
- [Player Events](#player-events)
- [World Events](#world-events)
- [Custom Events](#custom-events)
- [Async Event Handling](#async-event-handling)

---

## Event Base Class

Base class for all events in the system.

### Class Declaration

```java
public abstract class Event {
    private final long timestamp;
    private final String eventName;
    
    protected Event() {
        this.timestamp = System.currentTimeMillis();
        this.eventName = getClass().getSimpleName();
    }
    
    public long getTimestamp();
    public String getEventName();
    public boolean isAsynchronous();
}
```

### Methods

#### `getTimestamp()`

Returns when the event was created.

```java
Event event = new QuestCompletedEvent(player, quest);
long timestamp = event.getTimestamp();
logger.info("Event created at: {}", Instant.ofEpochMilli(timestamp));
```

**Returns:** Milliseconds since epoch  
**Uses:** Event ordering, debugging, analytics

---

#### `getEventName()`

Returns the event class name.

```java
String name = event.getEventName(); // "QuestCompletedEvent"
```

**Returns:** Simple class name  
**Uses:** Logging, debugging

---

#### `isAsynchronous()`

Returns whether event is fired asynchronously.

```java
if (event.isAsynchronous()) {
    logger.debug("Event fired on async thread");
}
```

**Returns:** `true` if event is async  
**Default:** `false` (synchronous)  
**Thread Safety:** Async events must be thread-safe

---

### Cancellable Interface

Events that can be cancelled implement this interface.

```java
public interface Cancellable {
    boolean isCancelled();
    void setCancelled(boolean cancelled);
}
```

### Example Event

```java
public class QuestStartedEvent extends Event implements Cancellable {
    private final Player player;
    private final Quest quest;
    private boolean cancelled = false;
    
    public QuestStartedEvent(Player player, Quest quest) {
        super();
        this.player = player;
        this.quest = quest;
    }
    
    public Player getPlayer() {
        return player;
    }
    
    public Quest getQuest() {
        return quest;
    }
    
    @Override
    public boolean isCancelled() {
        return cancelled;
    }
    
    @Override
    public void setCancelled(boolean cancelled) {
        this.cancelled = cancelled;
    }
}
```

---

## EventBus

Central event publication and subscription system.

### Class Declaration

```java
public interface EventBus {
    // Synchronous posting
    <T extends Event> void post(T event);
    
    // Asynchronous posting
    <T extends Event> CompletableFuture<T> postAsync(T event);
    
    // Subscription
    <T extends Event> void subscribe(Class<T> eventType, EventListener<T> listener);
    <T extends Event> void subscribe(Class<T> eventType, EventPriority priority, EventListener<T> listener);
    <T extends Event> void subscribe(Class<T> eventType, EventPriority priority, boolean ignoreCancelled, EventListener<T> listener);
    
    // Unsubscription
    <T extends Event> void unsubscribe(Class<T> eventType, EventListener<T> listener);
    void unsubscribeAll(Object listener);
    
    // Queries
    int getSubscriberCount(Class<? extends Event> eventType);
    Collection<Class<? extends Event>> getRegisteredEventTypes();
}
```

### Methods

#### `post(T event)`

Publishes an event synchronously.

```java
EventBus eventBus = platform.getEventBus();

// Fire quest completed event
QuestCompletedEvent event = new QuestCompletedEvent(player, quest);
eventBus.post(event);

// Check if cancelled
if (event.isCancelled()) {
    logger.info("Quest completion was cancelled by a listener");
    return;
}

// Continue with quest completion
giveRewards(player, quest);
```

**Parameters:**
- `event` - Event instance to publish

**Execution:** Synchronous, blocks until all listeners complete  
**Order:** Listeners execute in priority order  
**Thread Safety:** Must be called on main thread (unless event is async)  
**Exception Handling:** Listener exceptions logged but don't stop propagation

---

#### `postAsync(T event)`

Publishes an event asynchronously.

```java
CompletableFuture<QuestCompletedEvent> future = 
    eventBus.postAsync(new QuestCompletedEvent(player, quest));

future.thenAccept(event -> {
    if (!event.isCancelled()) {
        logger.info("Quest completed asynchronously");
    }
}).exceptionally(throwable -> {
    logger.error("Error processing async event", throwable);
    return null;
});
```

**Parameters:**
- `event` - Event instance to publish

**Returns:** `CompletableFuture<T>` completed after all listeners execute  
**Execution:** Asynchronous on event executor thread pool  
**Thread Safety:** Event and listeners must be thread-safe

---

#### `subscribe(Class<T> eventType, EventListener<T> listener)`

Subscribes to events with normal priority.

```java
eventBus.subscribe(QuestStartedEvent.class, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    logger.info("Player {} started quest {}", player.getName(), quest.getId());
});
```

**Parameters:**
- `eventType` - Event class to listen for
- `listener` - Functional listener interface

**Priority:** `EventPriority.NORMAL` (default)  
**Thread Safety:** Thread-safe

---

#### `subscribe(Class<T> eventType, EventPriority priority, EventListener<T> listener)`

Subscribes with specific priority.

```java
// High priority validator (runs first)
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGH, event -> {
    if (!validateCompletion(event.getPlayer(), event.getQuest())) {
        event.setCancelled(true);
    }
});

// Normal priority handler
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.NORMAL, event -> {
    if (!event.isCancelled()) {
        giveRewards(event.getPlayer(), event.getQuest());
    }
});

// Monitor priority logger (runs last, observes only)
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.MONITOR, event -> {
    logger.info("Quest {} completed by {}, cancelled: {}", 
               event.getQuest().getId(), 
               event.getPlayer().getName(),
               event.isCancelled());
});
```

**Parameters:**
- `eventType` - Event class to listen for
- `priority` - Execution priority
- `listener` - Functional listener interface

**See Also:** [EventPriority](#eventpriority)

---

#### `subscribe(Class<T> eventType, EventPriority priority, boolean ignoreCancelled, EventListener<T> listener)`

Subscribes with priority and cancellation handling.

```java
// This listener won't be called if event is cancelled
eventBus.subscribe(
    QuestCompletedEvent.class, 
    EventPriority.NORMAL, 
    true,  // ignoreCancelled = true
    event -> {
        // Only called if event not cancelled
        giveRewards(event.getPlayer(), event.getQuest());
    }
);
```

**Parameters:**
- `eventType` - Event class to listen for
- `priority` - Execution priority
- `ignoreCancelled` - Skip if event is cancelled
- `listener` - Functional listener interface

**Behavior:** If `ignoreCancelled` is `true`, listener skipped when `event.isCancelled()` is `true`

---

#### `unsubscribe(Class<T> eventType, EventListener<T> listener)`

Removes a specific listener.

```java
EventListener<QuestCompletedEvent> listener = event -> {
    // Handle event
};

// Subscribe
eventBus.subscribe(QuestCompletedEvent.class, listener);

// Later, unsubscribe
eventBus.unsubscribe(QuestCompletedEvent.class, listener);
```

**Parameters:**
- `eventType` - Event class
- `listener` - Listener to remove

**Thread Safety:** Thread-safe

---

#### `unsubscribeAll(Object listener)`

Removes all listeners registered by an object.

```java
public class QuestModule {
    public void onEnable() {
        eventBus.subscribe(QuestStartedEvent.class, this::onQuestStarted);
        eventBus.subscribe(QuestCompletedEvent.class, this::onQuestCompleted);
        eventBus.subscribe(ObjectiveProgressEvent.class, this::onObjectiveProgress);
    }
    
    public void onDisable() {
        // Unsubscribe all listeners registered by this module
        eventBus.unsubscribeAll(this);
    }
}
```

**Parameters:**
- `listener` - Object that registered listeners

**Thread Safety:** Thread-safe

---

### Example: Event Bus Usage

```java
public class QuestEventManager {
    private final EventBus eventBus;
    private final QuestManager questManager;
    private final Logger logger;
    
    public QuestEventManager(Platform platform) {
        this.eventBus = platform.getEventBus();
        this.questManager = platform.getServiceRegistry().getService(QuestManager.class);
        this.logger = platform.getLogger(QuestEventManager.class);
    }
    
    public void registerListeners() {
        // Validation (highest priority, runs first)
        eventBus.subscribe(
            QuestStartedEvent.class, 
            EventPriority.HIGHEST, 
            this::validateQuestStart
        );
        
        // Pre-processing (high priority)
        eventBus.subscribe(
            QuestCompletedEvent.class, 
            EventPriority.HIGH, 
            this::preProcessCompletion
        );
        
        // Main processing (normal priority)
        eventBus.subscribe(
            QuestCompletedEvent.class, 
            EventPriority.NORMAL, 
            true,  // Ignore if cancelled
            this::processCompletion
        );
        
        // Post-processing (low priority)
        eventBus.subscribe(
            QuestCompletedEvent.class, 
            EventPriority.LOW, 
            true,
            this::postProcessCompletion
        );
        
        // Logging (monitor priority, runs last)
        eventBus.subscribe(
            QuestCompletedEvent.class, 
            EventPriority.MONITOR, 
            this::logCompletion
        );
    }
    
    private void validateQuestStart(QuestStartedEvent event) {
        Player player = event.getPlayer();
        Quest quest = event.getQuest();
        
        // Check prerequisites
        if (!questManager.canStart(player, quest.getId())) {
            event.setCancelled(true);
            player.sendMessage("§cYou don't meet the requirements for this quest.");
            return;
        }
        
        // Check active quest limit
        int activeCount = questManager.getActiveQuests(player).size();
        if (activeCount >= 5) {
            event.setCancelled(true);
            player.sendMessage("§cYou have too many active quests!");
        }
    }
    
    private void processCompletion(QuestCompletedEvent event) {
        Player player = event.getPlayer();
        Quest quest = event.getQuest();
        
        // Give rewards
        for (QuestReward reward : event.getRewards()) {
            reward.apply(player);
        }
        
        // Update statistics
        updatePlayerStats(player, quest);
        
        // Unlock follow-up quests
        unlockFollowUpQuests(player, quest);
    }
    
    private void logCompletion(QuestCompletedEvent event) {
        logger.info("Quest completion event: quest={}, player={}, cancelled={}", 
                   event.getQuest().getId(),
                   event.getPlayer().getName(),
                   event.isCancelled());
    }
}
```

---

## EventListener

Functional interface for event listeners.

### Interface Declaration

```java
@FunctionalInterface
public interface EventListener<T extends Event> {
    void onEvent(T event);
}
```

### Usage

```java
// Lambda expression
eventBus.subscribe(QuestStartedEvent.class, event -> {
    logger.info("Quest started: {}", event.getQuest().getId());
});

// Method reference
eventBus.subscribe(QuestCompletedEvent.class, this::handleQuestCompleted);

// Anonymous class
eventBus.subscribe(NPCInteractEvent.class, new EventListener<NPCInteractEvent>() {
    @Override
    public void onEvent(NPCInteractEvent event) {
        handleNPCInteraction(event);
    }
});
```

---

## EventPriority

Defines execution order for event listeners.

### Enum Declaration

```java
public enum EventPriority {
    LOWEST(0),
    LOW(1),
    NORMAL(2),
    HIGH(3),
    HIGHEST(4),
    MONITOR(5);
    
    private final int slot;
    
    EventPriority(int slot) {
        this.slot = slot;
    }
    
    public int getSlot() {
        return slot;
    }
}
```

### Priority Levels

| Priority | Slot | Purpose | Can Cancel? | Can Modify? |
|----------|------|---------|-------------|-------------|
| LOWEST | 0 | Early cancellation, validation | Yes | Yes |
| LOW | 1 | Pre-processing | Yes | Yes |
| NORMAL | 2 | Main processing (default) | Yes | Yes |
| HIGH | 3 | Post-processing | Yes | Yes |
| HIGHEST | 4 | Final modifications | Yes | Yes |
| MONITOR | 5 | Observation only | No* | No* |

\* MONITOR should observe only, not cancel or modify events

### Best Practices

```java
// LOWEST - Early validation and cancellation
eventBus.subscribe(QuestStartedEvent.class, EventPriority.LOWEST, event -> {
    if (!hasPermission(event.getPlayer())) {
        event.setCancelled(true);
    }
});

// LOW - Pre-processing
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.LOW, event -> {
    prepareRewards(event);
});

// NORMAL - Main processing
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.NORMAL, event -> {
    giveRewards(event.getPlayer(), event.getRewards());
});

// HIGH - Post-processing
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGH, event -> {
    updateAchievements(event.getPlayer(), event.getQuest());
});

// HIGHEST - Final modifications
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGHEST, event -> {
    // Add bonus rewards based on all previous processing
    if (shouldGiveBonus(event)) {
        event.addReward(createBonusReward());
    }
});

// MONITOR - Logging and statistics (observe only!)
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.MONITOR, event -> {
    logger.info("Quest completed: {}", event.getQuest().getId());
    analytics.trackQuestCompletion(event);
});
```

---

## Quest Events

Events related to quest system operations.

### QuestStartedEvent

Fired when a player starts a quest.

```java
public class QuestStartedEvent extends Event implements Cancellable {
    private final Player player;
    private final Quest quest;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** Before quest is added to player's active quests  
**Cancellable:** Yes  
**Effect if Cancelled:** Quest is not started

**Example:**
```java
eventBus.subscribe(QuestStartedEvent.class, EventPriority.HIGH, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    
    // Check custom requirements
    if (!player.hasPermission("quests.category." + quest.getCategory().name().toLowerCase())) {
        event.setCancelled(true);
        player.sendMessage("§cYou don't have permission for this quest category!");
        return;
    }
    
    // Announce quest start
    player.sendMessage("§6§lQuest Started: §f" + quest.getName());
    player.playSound(player.getLocation(), Sound.QUEST_START, 1.0f, 1.0f);
});
```

---

### QuestCompletedEvent

Fired when a quest is completed.

```java
public class QuestCompletedEvent extends Event implements Cancellable {
    private final Player player;
    private final Quest quest;
    private final List<QuestReward> rewards;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public List<QuestReward> getRewards() { return rewards; }
    public void addReward(QuestReward reward) { rewards.add(reward); }
    public void removeReward(QuestReward reward) { rewards.remove(reward); }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** Before rewards are given and quest is marked complete  
**Cancellable:** Yes  
**Effect if Cancelled:** Quest completion is aborted, rewards not given

**Example:**
```java
eventBus.subscribe(QuestCompletedEvent.class, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    
    // Add bonus rewards for first completion
    String metaKey = "first_completion_" + quest.getId();
    if (!player.hasMetadata(metaKey)) {
        event.addReward(QuestReward.experience(100));
        event.addReward(QuestReward.money(500));
        player.setMetadata(metaKey, true);
        player.sendMessage("§6§lFirst Time Bonus!");
    }
    
    // Broadcast epic quest completions
    if (quest.getCategory() == QuestCategory.EPIC) {
        String message = String.format("§6%s §ehas completed §6%s§e!", 
                                      player.getName(), quest.getName());
        platform.getProvider().getOnlinePlayers().forEach(p -> p.sendMessage(message));
    }
    
    // Update leaderboards
    updateQuestLeaderboard(player, quest);
});
```

---

### ObjectiveProgressEvent

Fired when objective progress is updated.

```java
public class ObjectiveProgressEvent extends Event {
    private final Player player;
    private final Quest quest;
    private final QuestObjective objective;
    private final int previousProgress;
    private final int newProgress;
    private final boolean completed;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public QuestObjective getObjective() { return objective; }
    public int getPreviousProgress() { return previousProgress; }
    public int getNewProgress() { return newProgress; }
    public int getProgressDelta() { return newProgress - previousProgress; }
    public boolean isCompleted() { return completed; }
}
```

**When Fired:** After objective progress is updated  
**Cancellable:** No  
**Use Cases:** Notifications, UI updates, statistics

**Example:**
```java
eventBus.subscribe(ObjectiveProgressEvent.class, event -> {
    Player player = event.getPlayer();
    QuestObjective objective = event.getObjective();
    
    if (event.isCompleted()) {
        // Objective just completed
        player.sendMessage("§a§l✓ " + objective.getDescription());
        player.playSound(player.getLocation(), Sound.OBJECTIVE_COMPLETE, 1.0f, 1.0f);
        
        // Show next objective
        Quest quest = event.getQuest();
        QuestProgress progress = questManager.getProgress(player, quest.getId());
        for (QuestObjective next : quest.getObjectives()) {
            if (!progress.isObjectiveComplete(next.getId())) {
                player.sendMessage("§eNext: §f" + next.getDescription());
                break;
            }
        }
    } else {
        // Progress updated but not complete
        player.sendActionBar(String.format(
            "§e%s: §a%d§7/§a%d",
            objective.getDescription(),
            event.getNewProgress(),
            objective.getAmount()
        ));
    }
});
```

---

### QuestAbandonedEvent

Fired when a player abandons a quest.

```java
public class QuestAbandonedEvent extends Event implements Cancellable {
    private final Player player;
    private final Quest quest;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** Before quest is removed from active quests  
**Cancellable:** Yes  
**Effect if Cancelled:** Quest is not abandoned

**Example:**
```java
eventBus.subscribe(QuestAbandonedEvent.class, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    
    // Require confirmation for important quests
    if (quest.getCategory() == QuestCategory.MAIN) {
        if (!player.hasMetadata("confirm_abandon_" + quest.getId())) {
            event.setCancelled(true);
            player.sendMessage("§cType /quest abandon confirm to abandon this main quest.");
            player.setMetadata("confirm_abandon_" + quest.getId(), true);
            
            // Clear confirmation after 30 seconds
            scheduler.runTaskLater(() -> {
                player.removeMetadata("confirm_abandon_" + quest.getId());
            }, 600); // 30 seconds
            return;
        }
    }
    
    player.sendMessage("§eQuest abandoned: " + quest.getName());
});
```

---

## NPC Events

Events related to NPC interactions.

### NPCSpawnEvent

Fired when an NPC spawns.

```java
public class NPCSpawnEvent extends Event implements Cancellable {
    private final NPC npc;
    private Location location;
    private boolean cancelled = false;
    
    public NPC getNPC() { return npc; }
    public Location getLocation() { return location; }
    public void setLocation(Location location) { this.location = location; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** Before NPC entity is created in world  
**Cancellable:** Yes  
**Modifiable:** Location can be changed

**Example:**
```java
eventBus.subscribe(NPCSpawnEvent.class, event -> {
    NPC npc = event.getNPC();
    Location location = event.getLocation();
    
    // Prevent NPCs from spawning in certain areas
    if (isRestrictedArea(location)) {
        event.setCancelled(true);
        logger.warn("Prevented NPC {} from spawning in restricted area", npc.getId());
        return;
    }
    
    // Adjust spawn location if blocked
    if (!location.getBlock().isPassable()) {
        Location adjusted = findNearestPassableLocation(location);
        event.setLocation(adjusted);
        logger.debug("Adjusted NPC spawn location from {} to {}", location, adjusted);
    }
    
    logger.info("NPC {} spawning at {}", npc.getName(), location);
});
```

---

### PlayerInteractNPCEvent

Fired when a player interacts with an NPC.

```java
public class PlayerInteractNPCEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final InteractionType interactionType;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public InteractionType getInteractionType() { return interactionType; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** When player right-clicks or interacts with NPC  
**Cancellable:** Yes  
**Effect if Cancelled:** Default interaction (dialog) doesn't trigger

**Example:**
```java
eventBus.subscribe(PlayerInteractNPCEvent.class, EventPriority.HIGH, event -> {
    Player player = event.getPlayer();
    NPC npc = event.getNPC();
    
    // Custom quest giver interaction
    if (npc.getMetadata("quest_giver") == Boolean.TRUE) {
        event.setCancelled(true); // Prevent default dialog
        
        @SuppressWarnings("unchecked")
        List<String> questIds = (List<String>) npc.getMetadata("available_quests");
        
        if (questIds == null || questIds.isEmpty()) {
            player.sendMessage("§e[" + npc.getName() + "] §fI have no quests for you right now.");
            return;
        }
        
        // Show available quests
        Window questWindow = createQuestListWindow(player, npc, questIds);
        questWindow.open();
    }
    
    // Shop keeper interaction
    else if (npc.getMetadata("shop_keeper") == Boolean.TRUE) {
        event.setCancelled(true);
        
        if (event.getInteractionType() == InteractionType.SHIFT_RIGHT_CLICK) {
            showShopInfo(player, npc);
        } else {
            openShop(player, npc);
        }
    }
});
```

---

### DialogStartEvent

Fired when dialog starts between player and NPC.

```java
public class DialogStartEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final DialogTree dialogTree;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public DialogTree getDialogTree() { return dialogTree; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(DialogStartEvent.class, event -> {
    Player player = event.getPlayer();
    NPC npc = event.getNPC();
    
    // Make NPC look at player
    npc.lookAt(player);
    
    // Play greeting animation
    npc.playAnimation(NPCAnimation.WAVE);
    
    logger.debug("Dialog started between {} and {}", player.getName(), npc.getName());
});
```

---

### DialogChoiceEvent

Fired when player selects a dialog choice.

```java
public class DialogChoiceEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final DialogNode currentNode;
    private final DialogChoice choice;
    private String nextNodeId;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public DialogNode getCurrentNode() { return currentNode; }
    public DialogChoice getChoice() { return choice; }
    public String getNextNodeId() { return nextNodeId; }
    public void setNextNodeId(String nextNodeId) { this.nextNodeId = nextNodeId; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**When Fired:** After choice is selected, before navigation  
**Cancellable:** Yes  
**Modifiable:** Next node ID can be changed

**Example:**
```java
eventBus.subscribe(DialogChoiceEvent.class, event -> {
    Player player = event.getPlayer();
    DialogChoice choice = event.getChoice();
    
    logger.info("Player {} selected choice: {}", player.getName(), choice.getText());
    
    // Redirect based on player state
    if (choice.getText().contains("Quest")) {
        boolean hasActiveQuests = !questManager.getActiveQuests(player).isEmpty();
        if (hasActiveQuests) {
            event.setNextNodeId("active_quests");
        } else {
            event.setNextNodeId("no_active_quests");
        }
    }
});
```

---

## Player Events

Events related to player actions.

### PlayerJoinEvent

Fired when player joins the server.

```java
public class PlayerJoinEvent extends Event {
    private final Player player;
    private String joinMessage;
    
    public Player getPlayer() { return player; }
    public String getJoinMessage() { return joinMessage; }
    public void setJoinMessage(String message) { this.joinMessage = message; }
}
```

**Example:**
```java
eventBus.subscribe(PlayerJoinEvent.class, event -> {
    Player player = event.getPlayer();
    
    // Load quest progress
    questManager.loadProgress(player);
    
    // Check for completed quests while offline
    checkOfflineQuestCompletions(player);
    
    // Set custom join message
    int questCount = questManager.getCompletedQuests(player).size();
    event.setJoinMessage(String.format("§6%s §ejoined (§6%d quests§e completed)", 
                                      player.getName(), questCount));
});
```

---

### PlayerQuitEvent

Fired when player leaves the server.

```java
public class PlayerQuitEvent extends Event {
    private final Player player;
    private String quitMessage;
    
    public Player getPlayer() { return player; }
    public String getQuitMessage() { return quitMessage; }
    public void setQuitMessage(String message) { this.quitMessage = message; }
}
```

**Example:**
```java
eventBus.subscribe(PlayerQuitEvent.class, event -> {
    Player player = event.getPlayer();
    
    // Save quest progress
    questManager.saveProgress(player);
    
    // Clean up active dialogs
    DialogState dialogState = npcManager.getDialogState(player);
    if (dialogState != null) {
        npcManager.endDialog(player);
    }
    
    logger.info("Saved data for player: {}", player.getName());
});
```

---

### PlayerMoveEvent

Fired when player moves.

```java
public class PlayerMoveEvent extends Event implements Cancellable {
    private final Player player;
    private final Location from;
    private Location to;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public Location getFrom() { return from; }
    public Location getTo() { return to; }
    public void setTo(Location to) { this.to = to; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(PlayerMoveEvent.class, event -> {
    Player player = event.getPlayer();
    Location to = event.getTo();
    
    // Check location objectives
    Collection<QuestProgress> activeQuests = questManager.getActiveQuests(player);
    for (QuestProgress progress : activeQuests) {
        Quest quest = progress.getQuest();
        
        for (QuestObjective objective : quest.getObjectives()) {
            if (objective.getType() == ObjectiveType.LOCATION 
                && !progress.isObjectiveComplete(objective.getId())) {
                
                if (isInTargetArea(to, objective)) {
                    questManager.updateObjective(player, quest.getId(), objective.getId(), 1);
                }
            }
        }
    }
});
```

---

### PlayerDeathEvent

Fired when player dies.

```java
public class PlayerDeathEvent extends Event {
    private final Player player;
    private final Entity killer;
    private String deathMessage;
    private boolean keepInventory = false;
    
    public Player getPlayer() { return player; }
    public Entity getKiller() { return killer; }
    public String getDeathMessage() { return deathMessage; }
    public void setDeathMessage(String message) { this.deathMessage = message; }
    public boolean getKeepInventory() { return keepInventory; }
    public void setKeepInventory(boolean keep) { this.keepInventory = keep; }
}
```

**Example:**
```java
eventBus.subscribe(PlayerDeathEvent.class, event -> {
    Player player = event.getPlayer();
    
    // Check death objectives
    Collection<QuestProgress> activeQuests = questManager.getActiveQuests(player);
    for (QuestProgress progress : activeQuests) {
        Quest quest = progress.getQuest();
        
        // Fail quests that don't allow death
        if (quest.getMetadata("fail_on_death") == Boolean.TRUE) {
            questManager.abandonQuest(player, quest.getId());
            player.sendMessage("§cQuest failed: " + quest.getName());
        }
    }
});
```

---

## World Events

Events related to world interactions.

### BlockBreakEvent

Fired when a block is broken.

```java
public class BlockBreakEvent extends Event implements Cancellable {
    private final Player player;
    private final Block block;
    private int experience;
    private boolean cancelled = false;
    
    public Player getPlayer() { return player; }
    public Block getBlock() { return block; }
    public int getExperience() { return experience; }
    public void setExperience(int xp) { this.experience = xp; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(BlockBreakEvent.class, event -> {
    Player player = event.getPlayer();
    Block block = event.getBlock();
    
    // Check gathering objectives
    Collection<QuestProgress> activeQuests = questManager.getActiveQuests(player);
    for (QuestProgress progress : activeQuests) {
        Quest quest = progress.getQuest();
        
        for (QuestObjective objective : quest.getObjectives()) {
            if (objective.getType() == ObjectiveType.COLLECT_ITEMS 
                && objective.getTarget().equals(block.getType().name().toLowerCase())
                && !progress.isObjectiveComplete(objective.getId())) {
                
                questManager.updateObjective(player, quest.getId(), objective.getId(), 1);
            }
        }
    }
});
```

---

### EntityDeathEvent

Fired when an entity dies.

```java
public class EntityDeathEvent extends Event {
    private final Entity entity;
    private final Player killer;
    private List<ItemStack> drops;
    private int experience;
    
    public Entity getEntity() { return entity; }
    public Player getKiller() { return killer; }
    public List<ItemStack> getDrops() { return drops; }
    public void setDrops(List<ItemStack> drops) { this.drops = drops; }
    public int getExperience() { return experience; }
    public void setExperience(int xp) { this.experience = xp; }
}
```

**Example:**
```java
eventBus.subscribe(EntityDeathEvent.class, event -> {
    Player killer = event.getKiller();
    if (killer == null) return;
    
    Entity entity = event.getEntity();
    String mobType = entity.getType().name().toLowerCase();
    
    // Check kill objectives
    Collection<QuestProgress> activeQuests = questManager.getActiveQuests(killer);
    for (QuestProgress progress : activeQuests) {
        Quest quest = progress.getQuest();
        
        for (QuestObjective objective : quest.getObjectives()) {
            if (objective.getType() == ObjectiveType.KILL_MOBS 
                && objective.getTarget().equals(mobType)
                && !progress.isObjectiveComplete(objective.getId())) {
                
                questManager.updateObjective(killer, quest.getId(), objective.getId(), 1);
            }
        }
    }
});
```

---

## Custom Events

Creating custom events for your plugin.

### Simple Custom Event

```java
public class RewardClaimedEvent extends Event {
    private final Player player;
    private final QuestReward reward;
    
    public RewardClaimedEvent(Player player, QuestReward reward) {
        super();
        this.player = player;
        this.reward = reward;
    }
    
    public Player getPlayer() {
        return player;
    }
    
    public QuestReward getReward() {
        return reward;
    }
}

// Fire the event
RewardClaimedEvent event = new RewardClaimedEvent(player, reward);
eventBus.post(event);

// Listen for the event
eventBus.subscribe(RewardClaimedEvent.class, event -> {
    logger.info("Reward claimed: {} by {}", 
               event.getReward(), 
               event.getPlayer().getName());
});
```

---

### Cancellable Custom Event

```java
public class QuestChainProgressEvent extends Event implements Cancellable {
    private final Player player;
    private final QuestChain chain;
    private final Quest completedQuest;
    private final Quest nextQuest;
    private boolean cancelled = false;
    private boolean autoStart = true;
    
    public QuestChainProgressEvent(Player player, QuestChain chain, 
                                  Quest completed, Quest next) {
        super();
        this.player = player;
        this.chain = chain;
        this.completedQuest = completed;
        this.nextQuest = next;
    }
    
    public Player getPlayer() { return player; }
    public QuestChain getChain() { return chain; }
    public Quest getCompletedQuest() { return completedQuest; }
    public Quest getNextQuest() { return nextQuest; }
    public boolean shouldAutoStart() { return autoStart; }
    public void setAutoStart(boolean autoStart) { this.autoStart = autoStart; }
    
    @Override
    public boolean isCancelled() { return cancelled; }
    
    @Override
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}

// Fire and handle cancellation
QuestChainProgressEvent event = new QuestChainProgressEvent(
    player, chain, completedQuest, nextQuest
);
eventBus.post(event);

if (!event.isCancelled() && event.shouldAutoStart()) {
    questManager.startQuest(player, nextQuest.getId());
}
```

---

## Async Event Handling

### Async Event Example

```java
public class DataSyncEvent extends Event {
    private final Player player;
    private final Map<String, Object> data;
    
    public DataSyncEvent(Player player, Map<String, Object> data) {
        super();
        this.player = player;
        this.data = data;
    }
    
    @Override
    public boolean isAsynchronous() {
        return true; // Mark as async event
    }
    
    public Player getPlayer() { return player; }
    public Map<String, Object> getData() { return data; }
}

// Fire asynchronously
CompletableFuture<DataSyncEvent> future = 
    eventBus.postAsync(new DataSyncEvent(player, data));

future.thenAccept(event -> {
    logger.info("Data sync complete for {}", event.getPlayer().getName());
}).exceptionally(throwable -> {
    logger.error("Error during async event", throwable);
    return null;
});

// Listen to async event (listener must be thread-safe!)
eventBus.subscribe(DataSyncEvent.class, event -> {
    // This runs on async thread - must be thread-safe!
    synchronized (databaseLock) {
        saveToDatabase(event.getPlayer(), event.getData());
    }
});
```

---

### Async Processing with Sync Callback

```java
public class QuestService {
    public void processQuestCompletionAsync(Player player, Quest quest) {
        // Start async processing
        CompletableFuture.runAsync(() -> {
            // Heavy processing on async thread
            calculateStatistics(player, quest);
            updateLeaderboards(player, quest);
            saveToDatabase(player, quest);
        }).thenRunAsync(() -> {
            // Back on main thread for game modifications
            QuestCompletedEvent event = new QuestCompletedEvent(player, quest);
            eventBus.post(event);
            
            if (!event.isCancelled()) {
                giveRewards(player, event.getRewards());
            }
        }, mainThreadExecutor);
    }
}
```

---

## See Also

- [Platform Core API](platform-core.md) - EventBus and Platform services
- [Quest Framework API](framework-quest.md) - Quest events
- [NPC Framework API](framework-npc.md) - NPC and dialog events
- [UI Framework API](framework-ui.md) - UI event handling
