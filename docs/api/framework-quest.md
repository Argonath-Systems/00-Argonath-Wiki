# Quest Framework API Reference

**Package:** `com.argonathsystems.framework.quest`  
**Module ID:** `quest`  
**Version:** 1.0.0  
**Dependencies:** `core`, `config`, `storage`, `objective`

## Overview

The Quest Framework provides a complete quest system with support for objectives, rewards, chains, prerequisites, and progress tracking. It features a fluent builder API, event-driven architecture, and extensible objective system.

## Table of Contents

- [Quest](#quest)
- [QuestBuilder](#questbuilder)
- [QuestRegistry](#questregistry)
- [QuestManager](#questmanager)
- [QuestProgress](#questprogress)
- [QuestObjective](#questobjective)
- [QuestReward](#questreward)
- [QuestChain](#questchain)
- [Quest Events](#quest-events)

---

## Quest

Immutable quest definition containing metadata, objectives, and rewards.

### Class Declaration

```java
public interface Quest {
    String getId();
    String getName();
    String getDescription();
    List<String> getLore();
    QuestCategory getCategory();
    List<QuestObjective> getObjectives();
    List<QuestReward> getRewards();
    List<String> getPrerequisites();
    String getChainId();
    boolean isRepeatable();
    Duration getCooldown();
    int getMinLevel();
    int getMaxLevel();
    Map<String, Object> getMetadata();
}
```

### Methods

#### `getId()`

Returns the unique quest identifier.

```java
Quest quest = questManager.getQuest("main_quest_1");
String id = quest.getId(); // "main_quest_1"
```

**Returns:** Unique quest ID  
**Immutability:** Immutable

---

#### `getName()`

Returns the display name of the quest.

```java
String name = quest.getName(); // "The Lost Artifact"
player.sendMessage("Quest: " + name);
```

**Returns:** Quest display name  
**Format:** Color codes supported (`&c`, `&l`, etc.)

---

#### `getDescription()`

Returns the quest description shown to players.

```java
String description = quest.getDescription();
// "Find the ancient artifact hidden in the mountain caves."
```

**Returns:** Quest description  
**Length:** Typically 1-3 sentences

---

#### `getLore()`

Returns additional quest lore/backstory lines.

```java
List<String> lore = quest.getLore();
for (String line : lore) {
    player.sendMessage(line);
}
```

**Returns:** Immutable list of lore lines  
**Format:** Color codes supported

---

#### `getObjectives()`

Returns all quest objectives.

```java
List<QuestObjective> objectives = quest.getObjectives();
for (QuestObjective objective : objectives) {
    logger.info("Objective: {}", objective.getDescription());
}
```

**Returns:** Immutable list of objectives  
**Order:** Objectives are ordered (display/completion order)

---

#### `getRewards()`

Returns all quest rewards.

```java
List<QuestReward> rewards = quest.getRewards();
for (QuestReward reward : rewards) {
    reward.apply(player);
}
```

**Returns:** Immutable list of rewards  
**Application:** Rewards applied when quest completes

---

#### `getPrerequisites()`

Returns quest IDs that must be completed before this quest.

```java
List<String> prerequisites = quest.getPrerequisites();
boolean canStart = prerequisites.stream()
    .allMatch(id -> questManager.isCompleted(player, id));
```

**Returns:** Immutable list of prerequisite quest IDs  
**Validation:** Prerequisites checked on quest start

---

#### `isRepeatable()`

Returns whether the quest can be repeated after completion.

```java
if (quest.isRepeatable()) {
    Duration cooldown = quest.getCooldown();
    logger.info("Quest can be repeated every {}", cooldown);
}
```

**Returns:** `true` if quest is repeatable  
**Default:** `false`

---

### Example Usage

```java
public class QuestDisplayService {
    public void showQuestInfo(Player player, Quest quest) {
        player.sendMessage("§6§l" + quest.getName());
        player.sendMessage("§7" + quest.getDescription());
        player.sendMessage("");
        
        // Show objectives
        player.sendMessage("§eObjectives:");
        QuestProgress progress = questManager.getProgress(player, quest.getId());
        for (QuestObjective objective : quest.getObjectives()) {
            boolean completed = progress.isObjectiveComplete(objective.getId());
            String status = completed ? "§a✓" : "§7○";
            player.sendMessage(status + " " + objective.getDescription());
        }
        
        // Show rewards
        player.sendMessage("");
        player.sendMessage("§eRewards:");
        for (QuestReward reward : quest.getRewards()) {
            player.sendMessage("§7- " + reward.getDescription());
        }
        
        // Show prerequisites
        if (!quest.getPrerequisites().isEmpty()) {
            player.sendMessage("");
            player.sendMessage("§eRequires:");
            for (String prereqId : quest.getPrerequisites()) {
                Quest prereq = questManager.getQuest(prereqId);
                boolean completed = questManager.isCompleted(player, prereqId);
                String status = completed ? "§a✓" : "§c✗";
                player.sendMessage(status + " " + prereq.getName());
            }
        }
    }
}
```

---

## QuestBuilder

Fluent builder for creating quest instances.

### Class Declaration

```java
public interface QuestBuilder {
    QuestBuilder id(String id);
    QuestBuilder name(String name);
    QuestBuilder description(String description);
    QuestBuilder lore(String... lines);
    QuestBuilder category(QuestCategory category);
    QuestBuilder objective(QuestObjective objective);
    QuestBuilder objectives(QuestObjective... objectives);
    QuestBuilder reward(QuestReward reward);
    QuestBuilder rewards(QuestReward... rewards);
    QuestBuilder prerequisite(String questId);
    QuestBuilder prerequisites(String... questIds);
    QuestBuilder chain(String chainId);
    QuestBuilder repeatable(boolean repeatable);
    QuestBuilder cooldown(Duration cooldown);
    QuestBuilder levelRange(int min, int max);
    QuestBuilder metadata(String key, Object value);
    Quest build();
}
```

### Methods

#### `id(String id)`

Sets the unique quest identifier.

```java
QuestBuilder builder = Quest.builder()
    .id("main_quest_1");
```

**Parameters:**
- `id` - Unique quest ID (required)

**Returns:** Builder for chaining  
**Validation:** Must be unique, alphanumeric with underscores/hyphens

---

#### `name(String name)`

Sets the quest display name.

```java
builder.name("§6The Lost Artifact");
```

**Parameters:**
- `name` - Display name (required, supports color codes)

**Returns:** Builder for chaining

---

#### `objective(QuestObjective objective)`

Adds a quest objective.

```java
builder.objective(
    QuestObjective.builder()
        .id("kill_zombies")
        .type(ObjectiveType.KILL_MOBS)
        .target("zombie")
        .amount(10)
        .description("Kill 10 zombies")
        .build()
);
```

**Parameters:**
- `objective` - Quest objective

**Returns:** Builder for chaining  
**Note:** Objectives are ordered

---

#### `reward(QuestReward reward)`

Adds a quest reward.

```java
builder.reward(QuestReward.experience(100))
       .reward(QuestReward.item(new ItemStack(Material.DIAMOND, 5)))
       .reward(QuestReward.money(1000));
```

**Parameters:**
- `reward` - Quest reward

**Returns:** Builder for chaining

---

#### `prerequisite(String questId)`

Adds a prerequisite quest.

```java
builder.prerequisite("tutorial_quest")
       .prerequisite("first_steps");
```

**Parameters:**
- `questId` - Required quest ID

**Returns:** Builder for chaining  
**Validation:** Prerequisite quest must exist

---

#### `repeatable(boolean repeatable)`

Sets whether quest can be repeated.

```java
builder.repeatable(true)
       .cooldown(Duration.ofHours(24));
```

**Parameters:**
- `repeatable` - Repeatability flag

**Returns:** Builder for chaining

---

#### `build()`

Constructs the immutable Quest instance.

```java
Quest quest = builder.build();
```

**Returns:** Immutable `Quest` instance  
**Throws:** `IllegalStateException` if required fields missing  
**Validation:** Validates all constraints

---

### Complete Example

```java
public class QuestDefinitions {
    public static Quest createMainQuest1() {
        return Quest.builder()
            .id("main_quest_1")
            .name("§6§lThe Lost Artifact")
            .description("Find the ancient artifact hidden in the mountain caves.")
            .lore(
                "§7Long ago, a powerful artifact was hidden",
                "§7in the depths of the mountain. Many have",
                "§7searched, but none have succeeded."
            )
            .category(QuestCategory.MAIN)
            
            // Objectives
            .objective(
                QuestObjective.builder()
                    .id("find_cave")
                    .type(ObjectiveType.LOCATION)
                    .target("mountain_cave")
                    .description("Find the mountain cave")
                    .build()
            )
            .objective(
                QuestObjective.builder()
                    .id("defeat_guardian")
                    .type(ObjectiveType.KILL_MOBS)
                    .target("cave_guardian")
                    .amount(1)
                    .description("Defeat the Cave Guardian")
                    .build()
            )
            .objective(
                QuestObjective.builder()
                    .id("retrieve_artifact")
                    .type(ObjectiveType.COLLECT_ITEMS)
                    .target("ancient_artifact")
                    .amount(1)
                    .description("Retrieve the Ancient Artifact")
                    .build()
            )
            
            // Rewards
            .reward(QuestReward.experience(500))
            .reward(QuestReward.money(1000))
            .reward(QuestReward.item(createArtifactItem()))
            .reward(QuestReward.title("§6Artifact Hunter"))
            
            // Prerequisites
            .prerequisite("tutorial_complete")
            
            // Settings
            .levelRange(10, 100)
            .repeatable(false)
            .metadata("recommended_level", 15)
            .metadata("estimated_time", "30 minutes")
            
            .build();
    }
}
```

---

## QuestRegistry

Central registry for quest definitions.

### Class Declaration

```java
public interface QuestRegistry {
    void registerQuest(Quest quest);
    void unregisterQuest(String questId);
    Quest getQuest(String questId);
    Collection<Quest> getAllQuests();
    Collection<Quest> getQuestsByCategory(QuestCategory category);
    Collection<Quest> getQuestsByChain(String chainId);
    Collection<Quest> getAvailableQuests(Player player);
    boolean isRegistered(String questId);
    void reload();
}
```

### Methods

#### `registerQuest(Quest quest)`

Registers a quest definition.

```java
QuestRegistry registry = questManager.getRegistry();
Quest quest = createMainQuest1();
registry.registerQuest(quest);
```

**Parameters:**
- `quest` - Quest to register

**Throws:** `IllegalArgumentException` if quest ID already registered  
**Thread Safety:** Thread-safe

---

#### `getQuest(String questId)`

Retrieves a quest by ID.

```java
Quest quest = registry.getQuest("main_quest_1");
if (quest == null) {
    logger.warn("Quest not found: main_quest_1");
}
```

**Parameters:**
- `questId` - Quest identifier

**Returns:** Quest instance or `null` if not found  
**Thread Safety:** Thread-safe

---

#### `getAvailableQuests(Player player)`

Returns quests available to a player based on level, prerequisites, etc.

```java
Collection<Quest> available = registry.getAvailableQuests(player);
for (Quest quest : available) {
    player.sendMessage("Available: " + quest.getName());
}
```

**Parameters:**
- `player` - Player to check

**Returns:** Collection of available quests  
**Filtering:** Checks level range, prerequisites, already completed (if not repeatable)

---

#### `reload()`

Reloads all quests from configuration files.

```java
registry.reload();
logger.info("Reloaded {} quests", registry.getAllQuests().size());
```

**Thread Safety:** Thread-safe  
**Side Effects:** Clears registry and reloads from disk

---

### Example Usage

```java
public class QuestCommand {
    private final QuestRegistry registry;
    
    public void handleListCommand(Player player, String category) {
        Collection<Quest> quests;
        
        if (category == null) {
            quests = registry.getAvailableQuests(player);
        } else {
            QuestCategory cat = QuestCategory.valueOf(category.toUpperCase());
            quests = registry.getQuestsByCategory(cat);
        }
        
        player.sendMessage("§6Available Quests:");
        for (Quest quest : quests) {
            int level = calculateQuestLevel(quest);
            player.sendMessage(String.format("§e[%d] §f%s", level, quest.getName()));
        }
    }
}
```

---

## QuestManager

High-level quest management and player progress tracking.

### Class Declaration

```java
public interface QuestManager {
    QuestRegistry getRegistry();
    boolean startQuest(Player player, String questId);
    boolean completeQuest(Player player, String questId);
    boolean abandonQuest(Player player, String questId);
    QuestProgress getProgress(Player player, String questId);
    Collection<QuestProgress> getActiveQuests(Player player);
    Collection<String> getCompletedQuests(Player player);
    boolean isActive(Player player, String questId);
    boolean isCompleted(Player player, String questId);
    boolean canStart(Player player, String questId);
    void updateObjective(Player player, String questId, String objectiveId, int amount);
    void saveProgress(Player player);
    void loadProgress(Player player);
}
```

### Methods

#### `startQuest(Player player, String questId)`

Starts a quest for a player.

```java
QuestManager manager = platform.getServiceRegistry()
    .getService(QuestManager.class);

boolean started = manager.startQuest(player, "main_quest_1");
if (started) {
    player.sendMessage("§aQuest started!");
} else {
    player.sendMessage("§cCannot start quest.");
}
```

**Parameters:**
- `player` - Player starting the quest
- `questId` - Quest identifier

**Returns:** `true` if quest started successfully  
**Events:** Fires `QuestStartedEvent`  
**Validation:** Checks prerequisites, level, active quest limit

---

#### `completeQuest(Player player, String questId)`

Completes a quest and gives rewards.

```java
boolean completed = manager.completeQuest(player, "main_quest_1");
if (completed) {
    player.sendMessage("§a§lQuest Complete!");
    // Rewards automatically applied
}
```

**Parameters:**
- `player` - Player completing the quest
- `questId` - Quest identifier

**Returns:** `true` if quest completed  
**Events:** Fires `QuestCompletedEvent`  
**Side Effects:** Applies rewards, updates statistics

---

#### `abandonQuest(Player player, String questId)`

Abandons an active quest.

```java
boolean abandoned = manager.abandonQuest(player, "main_quest_1");
if (abandoned) {
    player.sendMessage("Quest abandoned.");
}
```

**Parameters:**
- `player` - Player
- `questId` - Quest identifier

**Returns:** `true` if quest was abandoned  
**Events:** Fires `QuestAbandonedEvent`  
**Side Effects:** Clears progress

---

#### `getProgress(Player player, String questId)`

Retrieves quest progress for a player.

```java
QuestProgress progress = manager.getProgress(player, "main_quest_1");
if (progress != null) {
    int completion = progress.getCompletionPercentage();
    player.sendMessage("Progress: " + completion + "%");
}
```

**Parameters:**
- `player` - Player
- `questId` - Quest identifier

**Returns:** `QuestProgress` or `null` if not active  
**Thread Safety:** Returns immutable snapshot

---

#### `getActiveQuests(Player player)`

Returns all active quests for a player.

```java
Collection<QuestProgress> active = manager.getActiveQuests(player);
player.sendMessage("Active quests: " + active.size());
for (QuestProgress progress : active) {
    Quest quest = progress.getQuest();
    player.sendMessage("- " + quest.getName());
}
```

**Parameters:**
- `player` - Player

**Returns:** Collection of active quest progress  
**Thread Safety:** Thread-safe

---

#### `canStart(Player player, String questId)`

Checks if player can start a quest.

```java
boolean canStart = manager.canStart(player, "main_quest_1");
if (!canStart) {
    // Check specific reasons
    Quest quest = registry.getQuest("main_quest_1");
    if (player.getLevel() < quest.getMinLevel()) {
        player.sendMessage("Level too low!");
    }
}
```

**Parameters:**
- `player` - Player
- `questId` - Quest identifier

**Returns:** `true` if player meets all requirements  
**Checks:** Prerequisites, level, cooldown, active quest limit

---

#### `updateObjective(Player player, String questId, String objectiveId, int amount)`

Updates objective progress.

```java
// Player killed a zombie
manager.updateObjective(player, "main_quest_1", "kill_zombies", 1);
```

**Parameters:**
- `player` - Player
- `questId` - Quest identifier
- `objectiveId` - Objective identifier
- `amount` - Amount to add to progress

**Events:** Fires `ObjectiveProgressEvent`, may fire `QuestCompletedEvent`  
**Thread Safety:** Thread-safe

---

### Example Usage

```java
public class QuestService {
    private final QuestManager questManager;
    private final Platform platform;
    
    public void handlePlayerKillMob(Player player, String mobType) {
        Collection<QuestProgress> activeQuests = questManager.getActiveQuests(player);
        
        for (QuestProgress progress : activeQuests) {
            Quest quest = progress.getQuest();
            
            for (QuestObjective objective : quest.getObjectives()) {
                if (objective.getType() == ObjectiveType.KILL_MOBS 
                    && objective.getTarget().equals(mobType)
                    && !progress.isObjectiveComplete(objective.getId())) {
                    
                    questManager.updateObjective(
                        player, 
                        quest.getId(), 
                        objective.getId(), 
                        1
                    );
                    
                    // Get updated progress
                    QuestProgress updated = questManager.getProgress(player, quest.getId());
                    int current = updated.getObjectiveProgress(objective.getId());
                    int required = objective.getAmount();
                    
                    player.sendMessage(String.format(
                        "§e[Quest] §f%s: §a%d§7/§a%d",
                        objective.getDescription(),
                        current,
                        required
                    ));
                }
            }
        }
    }
}
```

---

## QuestProgress

Tracks player progress for a specific quest.

### Class Declaration

```java
public interface QuestProgress {
    Quest getQuest();
    Player getPlayer();
    QuestState getState();
    Instant getStartTime();
    Instant getCompletionTime();
    int getObjectiveProgress(String objectiveId);
    boolean isObjectiveComplete(String objectiveId);
    Map<String, Integer> getAllObjectiveProgress();
    int getCompletionPercentage();
    boolean isComplete();
}
```

### Methods

#### `getObjectiveProgress(String objectiveId)`

Returns current progress for an objective.

```java
QuestProgress progress = questManager.getProgress(player, "main_quest_1");
int zombiesKilled = progress.getObjectiveProgress("kill_zombies");
logger.info("Player has killed {} zombies", zombiesKilled);
```

**Parameters:**
- `objectiveId` - Objective identifier

**Returns:** Current progress amount  
**Default:** Returns 0 if objective not started

---

#### `isObjectiveComplete(String objectiveId)`

Checks if an objective is complete.

```java
boolean complete = progress.isObjectiveComplete("kill_zombies");
if (complete) {
    player.sendMessage("§a✓ Objective complete!");
}
```

**Parameters:**
- `objectiveId` - Objective identifier

**Returns:** `true` if objective is complete

---

#### `getCompletionPercentage()`

Returns overall quest completion percentage.

```java
int completion = progress.getCompletionPercentage();
player.sendActionBar(String.format("Quest: %d%%", completion));
```

**Returns:** Percentage from 0-100  
**Calculation:** Based on all objectives weighted equally

---

### Quest State Enum

```java
public enum QuestState {
    NOT_STARTED,
    ACTIVE,
    COMPLETED,
    FAILED,
    ABANDONED
}
```

---

## QuestObjective

Defines a quest objective/task.

### Class Declaration

```java
public interface QuestObjective {
    String getId();
    ObjectiveType getType();
    String getTarget();
    int getAmount();
    String getDescription();
    Map<String, Object> getMetadata();
    
    static QuestObjectiveBuilder builder() {
        return new QuestObjectiveBuilderImpl();
    }
}
```

### Objective Types

```java
public enum ObjectiveType {
    KILL_MOBS,          // Kill specific mob types
    COLLECT_ITEMS,      // Collect items
    LOCATION,           // Reach a location
    INTERACT_NPC,       // Talk to an NPC
    INTERACT_BLOCK,     // Interact with blocks
    CRAFT_ITEMS,        // Craft items
    DELIVER_ITEMS,      // Deliver items to NPC
    ESCORT,             // Escort an NPC
    EXPLORE,            // Explore areas
    CUSTOM              // Custom objective type
}
```

### Example Objectives

```java
// Kill objective
QuestObjective killObjective = QuestObjective.builder()
    .id("kill_zombies")
    .type(ObjectiveType.KILL_MOBS)
    .target("zombie")
    .amount(10)
    .description("Kill 10 zombies")
    .build();

// Collection objective
QuestObjective collectObjective = QuestObjective.builder()
    .id("collect_iron")
    .type(ObjectiveType.COLLECT_ITEMS)
    .target("iron_ingot")
    .amount(32)
    .description("Collect 32 iron ingots")
    .build();

// Location objective
QuestObjective locationObjective = QuestObjective.builder()
    .id("find_cave")
    .type(ObjectiveType.LOCATION)
    .target("mountain_cave")
    .description("Find the mountain cave")
    .metadata("x", 100)
    .metadata("y", 64)
    .metadata("z", -200)
    .metadata("radius", 10)
    .build();
```

---

## QuestReward

Defines quest rewards given on completion.

### Class Declaration

```java
public interface QuestReward {
    RewardType getType();
    String getDescription();
    void apply(Player player);
    
    // Factory methods
    static QuestReward experience(int amount);
    static QuestReward money(double amount);
    static QuestReward item(ItemStack item);
    static QuestReward items(ItemStack... items);
    static QuestReward command(String command);
    static QuestReward permission(String permission);
    static QuestReward title(String title);
    static QuestReward custom(Consumer<Player> rewardFunction);
}
```

### Reward Types

```java
public enum RewardType {
    EXPERIENCE,
    MONEY,
    ITEMS,
    COMMAND,
    PERMISSION,
    TITLE,
    CUSTOM
}
```

### Example Rewards

```java
Quest quest = Quest.builder()
    .id("example_quest")
    .name("Example Quest")
    
    // Experience reward
    .reward(QuestReward.experience(500))
    
    // Money reward
    .reward(QuestReward.money(1000.0))
    
    // Item rewards
    .reward(QuestReward.item(new ItemStack(Material.DIAMOND_SWORD)))
    .reward(QuestReward.items(
        new ItemStack(Material.DIAMOND, 5),
        new ItemStack(Material.EMERALD, 10)
    ))
    
    // Command reward (run as console)
    .reward(QuestReward.command("give {player} special_item 1"))
    
    // Permission reward
    .reward(QuestReward.permission("quests.vip"))
    
    // Title reward
    .reward(QuestReward.title("§6Quest Master"))
    
    // Custom reward
    .reward(QuestReward.custom(player -> {
        player.setHealth(player.getMaxHealth());
        player.sendMessage("§aHealth restored!");
    }))
    
    .build();
```

---

## QuestChain

Represents a series of linked quests.

### Class Declaration

```java
public interface QuestChain {
    String getId();
    String getName();
    String getDescription();
    List<String> getQuestIds();
    int getChainProgress(Player player);
    boolean isChainComplete(Player player);
    String getNextQuest(Player player);
}
```

### Example Usage

```java
public class QuestChainDefinitions {
    public static QuestChain createMainStoryChain() {
        return QuestChain.builder()
            .id("main_story")
            .name("§6§lMain Story")
            .description("Follow the main storyline")
            .addQuest("prologue")
            .addQuest("chapter_1")
            .addQuest("chapter_2")
            .addQuest("chapter_3")
            .addQuest("finale")
            .build();
    }
}

// Check chain progress
QuestChain chain = chainRegistry.getChain("main_story");
int progress = chain.getChainProgress(player); // Returns number of completed quests
String nextQuest = chain.getNextQuest(player); // Returns next uncompleted quest ID
```

---

## Quest Events

### QuestStartedEvent

Fired when a player starts a quest.

```java
public class QuestStartedEvent extends Event implements Cancellable {
    private final Player player;
    private final Quest quest;
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(QuestStartedEvent.class, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    
    // Validate custom requirements
    if (!player.hasPermission("quests.premium") && quest.getCategory() == QuestCategory.PREMIUM) {
        event.setCancelled(true);
        player.sendMessage("§cThis quest requires premium access!");
        return;
    }
    
    // Notify player
    player.sendMessage("§aStarted quest: §f" + quest.getName());
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
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public List<QuestReward> getRewards() { return rewards; }
    public void addReward(QuestReward reward) { rewards.add(reward); }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGH, event -> {
    Player player = event.getPlayer();
    Quest quest = event.getQuest();
    
    // Add bonus reward for first completion
    String key = "first_completion_" + quest.getId();
    if (!player.hasMetadata(key)) {
        event.addReward(QuestReward.experience(100));
        event.addReward(QuestReward.money(500));
        player.setMetadata(key, true);
        player.sendMessage("§6§lBonus reward for first completion!");
    }
    
    // Broadcast epic quest completions
    if (quest.getCategory() == QuestCategory.EPIC) {
        platform.getProvider().getOnlinePlayers().forEach(p -> 
            p.sendMessage(String.format("§6%s §ecompleted §6%s§e!", 
                         player.getName(), quest.getName()))
        );
    }
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
    public boolean isCompleted() { return completed; }
    public int getProgressDelta() { return newProgress - previousProgress; }
}
```

**Example:**
```java
eventBus.subscribe(ObjectiveProgressEvent.class, event -> {
    if (event.isCompleted()) {
        Player player = event.getPlayer();
        QuestObjective objective = event.getObjective();
        
        player.sendMessage("§a§l✓ Objective Complete: §f" + objective.getDescription());
        player.playSound(player.getLocation(), Sound.OBJECTIVE_COMPLETE, 1.0f, 1.0f);
        
        // Show next objective
        Quest quest = event.getQuest();
        QuestProgress progress = questManager.getProgress(player, quest.getId());
        
        for (QuestObjective nextObj : quest.getObjectives()) {
            if (!progress.isObjectiveComplete(nextObj.getId())) {
                player.sendMessage("§eNext: §f" + nextObj.getDescription());
                break;
            }
        }
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
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public Quest getQuest() { return quest; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

---

## See Also

- [Platform Core API](platform-core.md) - Core platform services
- [Event System API](events.md) - Complete event reference
- [Objective Framework API](framework-objective.md) - Objective system
- [NPC Framework API](framework-npc.md) - NPC integration for quest givers
