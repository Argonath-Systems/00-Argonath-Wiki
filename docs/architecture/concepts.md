# Core Concepts

Essential concepts and terminology for understanding Argonath Systems.

## Overview

Argonath Systems is built around several core concepts that form the foundation of the framework. Understanding these concepts is crucial for effective development.

## Platform Abstraction

### What is Platform Abstraction?

A design pattern that separates game-specific code from framework logic, allowing the same framework code to work across different game engines or platforms.

### Why Platform Abstraction?

**Benefits:**
- ✅ **Portability** - Framework works on multiple platforms
- ✅ **Testability** - Can test without running the game
- ✅ **Maintainability** - Changes isolated to adapters
- ✅ **Future-Proofing** - Easy to support new platforms

**Example:**
```java
// Platform-agnostic code
Player player = platform.getPlayer(uuid);
player.sendMessage("Hello!");

// Hytale adapter translates to:
// hytalePlayer.sendChatMessage(Text.of("Hello!"));
```

### Platform Layers

```
┌──────────────────────────────────────┐
│         Your Mod/Framework           │ (Platform-agnostic)
├──────────────────────────────────────┤
│         Platform SDK                 │ (Abstractions)
├──────────────────────────────────────┤
│         Platform Core                │ (Interfaces)
├──────────────────────────────────────┤
│         Adapter (Hytale)             │ (Game-specific)
├──────────────────────────────────────┤
│         Hytale Game Engine           │ (Game runtime)
└──────────────────────────────────────┘
```

---

## Registry Pattern

### What is a Registry?

A centralized storage and lookup system for game objects (quests, NPCs, items, etc.).

### Why Registries?

- **Centralized Management** - One place to find all objects
- **Type Safety** - Compile-time checking
- **Discoverability** - Easy to enumerate all registered objects
- **Lifecycle Control** - Manage initialization order

### Registry Example

```java
// Register a quest
QuestRegistry.register(myQuest);

// Retrieve by ID
Quest quest = QuestRegistry.get("my_quest_id");

// List all quests
Collection<Quest> allQuests = QuestRegistry.getAll();

// Check existence
boolean exists = QuestRegistry.contains("my_quest_id");
```

### Built-in Registries

- `QuestRegistry` - Quests
- `NPCRegistry` - NPCs
- `ObjectiveTypeRegistry` - Objective types
- `ConditionTypeRegistry` - Condition types
- `UIRegistry` - UI components

---

## Builder Pattern

### What is the Builder Pattern?

A fluent API for constructing complex objects step-by-step with readable, chainable method calls.

### Why Builders?

- ✅ **Readability** - Self-documenting code
- ✅ **Flexibility** - Optional parameters easy to handle
- ✅ **Validation** - Validate during build
- ✅ **Immutability** - Create immutable objects

### Builder Example

```java
Quest quest = QuestBuilder.create("epic_quest")
    .name("Epic Adventure")
    .description("An amazing journey!")
    .startCondition(Condition.level(10))
    .addObjective(
        Objective.builder()
            .type(ObjectiveType.KILL_ENTITY)
            .target("dragon")
            .count(1)
            .build()
    )
    .addReward(reward -> reward
        .gold(1000)
        .item("legendary_sword", 1)
    )
    .build();
```

**vs Traditional Constructor:**
```java
// Hard to read, easy to make mistakes
Quest quest = new Quest(
    "epic_quest",
    "Epic Adventure",
    "An amazing journey!",
    new LevelCondition(10),
    Arrays.asList(new KillObjective("dragon", 1)),
    new Reward(1000, Arrays.asList(new ItemStack("legendary_sword", 1))),
    null, // no prerequisites
    false, // not repeatable
    QuestCategory.MAIN // category
);
```

---

## Component System

### What are Components?

Modular, reusable pieces of functionality that can be attached to entities or objects.

### Component Lifecycle

```
Create → Initialize → Active → Dispose
```

### Component Example

```java
public class HealthComponent implements Component {
    private int health;
    private int maxHealth;
    
    @Override
    public void initialize() {
        health = maxHealth;
    }
    
    @Override
    public void update() {
        // Update logic each tick
    }
    
    @Override
    public void dispose() {
        // Cleanup
    }
    
    public void damage(int amount) {
        health = Math.max(0, health - amount);
    }
}

// Attach to entity
entity.addComponent(new HealthComponent(100));

// Retrieve component
HealthComponent health = entity.getComponent(HealthComponent.class);
health.damage(20);
```

### Component Benefits

- **Composition over Inheritance** - More flexible than class hierarchies
- **Reusability** - Use same component on different entities
- **Modularity** - Add/remove features dynamically

---

## Event-Driven Architecture

### What are Events?

Notifications that something happened, allowing decoupled communication between systems.

### Event Flow

```
Event Source → Event Bus → Event Listeners
```

### Event Example

```java
// Define event
public class QuestCompletedEvent extends Event {
    private final Quest quest;
    private final Player player;
    
    // ... constructor, getters
}

// Listen for event
EventBus.subscribe(QuestCompletedEvent.class, event -> {
    Quest quest = event.getQuest();
    Player player = event.getPlayer();
    
    player.sendMessage("Quest completed: " + quest.getName());
    // Give rewards, update stats, etc.
});

// Fire event
EventBus.fire(new QuestCompletedEvent(quest, player));
```

### Event Benefits

- **Decoupling** - Components don't need to know about each other
- **Extensibility** - Add new listeners without modifying sources
- **Debugging** - Easy to log all events

### Built-in Events

- `QuestStartedEvent`
- `QuestCompletedEvent`
- `ObjectiveProgressEvent`
- `NPCInteractEvent`
- `PlayerJoinEvent`
- Custom events via `Event` base class

---

## Condition System

### What are Conditions?

Predicates that evaluate to true/false based on game state, used for gating content.

### Condition Types

```java
// Simple condition
Condition.level(10)

// Composite conditions
Condition.and(
    Condition.level(5),
    Condition.hasQuest("tutorial")
)

Condition.or(
    Condition.hasItem("key_red"),
    Condition.hasItem("key_blue")
)

Condition.not(
    Condition.questCompleted("forbidden_quest")
)
```

### Common Use Cases

- **Quest Requirements** - "Must be level 10"
- **Dialog Options** - Show different text based on state
- **Item Access** - Unlock content conditionally
- **Time-Based** - Day/night, seasonal events

### Custom Conditions

```java
public class CustomCondition implements Condition {
    @Override
    public boolean test(Player player) {
        // Your custom logic
        return player.getPlaytime() > 3600; // 1 hour
    }
}
```

---

## Objective System

### What are Objectives?

Trackable goals with progress that players must complete.

### Objective Lifecycle

```
Create → Track Progress → Check Completion → Reward
```

### Objective Types

| Type | Description | Example |
|------|-------------|---------|
| `KILL_ENTITY` | Kill specific entities | Kill 10 goblins |
| `COLLECT_ITEM` | Gather items | Collect 5 apples |
| `TALK_TO_NPC` | Interact with NPC | Talk to Elder |
| `REACH_LOCATION` | Travel to area | Reach the castle |
| `CRAFT_ITEM` | Craft specific item | Craft iron sword |
| `USE_ITEM` | Use item X times | Use potion 3 times |
| `CUSTOM` | Custom logic | Any condition |

### Objective Progress

```java
Objective objective = Objective.builder()
    .type(ObjectiveType.KILL_ENTITY)
    .target("goblin")
    .count(10)
    .build();

// Track progress
ObjectiveProgress progress = new ObjectiveProgress(objective);
progress.increment(); // 1/10
progress.setProgress(5); // 5/10
boolean complete = progress.isComplete(); // false

// Get percentage
float percent = progress.getPercentage(); // 0.5 (50%)
```

---

## Storage and Persistence

### What is Storage?

Mechanisms for saving and loading data across game sessions.

### Storage Types

#### Player Storage
Per-player data (quest progress, settings)

```java
PlayerStorage<QuestProgress> storage = 
    PlayerStorage.create("quests", QuestProgress.class);

// Save
storage.save(player, questProgress);

// Load
QuestProgress progress = storage.load(player);
```

#### World Storage
Per-world data (spawns, state)

```java
WorldStorage<SpawnPoint> storage = 
    WorldStorage.create("spawns", SpawnPoint.class);
```

#### Global Storage
Server-wide data (leaderboards, economy)

```java
GlobalStorage<Economy> storage = 
    GlobalStorage.create("economy", Economy.class);
```

### Serialization

Data is automatically serialized to JSON:

```json
{
  "questId": "my_quest",
  "progress": 50,
  "completed": false,
  "objectives": [
    {"id": "obj1", "progress": 5}
  ]
}
```

---

## Accessor Pattern

### What are Accessors?

Safe wrappers for accessing potentially unavailable objects.

### Why Accessors?

Handle cases where objects may not exist:
- Player offline
- World unloaded
- Entity despawned

### Accessor Example

```java
// Instead of direct access (risky):
Player player = getPlayer(); // might be null
player.sendMessage("Hi"); // NullPointerException!

// Use accessor:
Accessor<Player> playerAccessor = Accessor.of(() -> getPlayer());

if (playerAccessor.isPresent()) {
    playerAccessor.get().sendMessage("Hi");
}

// Or with ifPresent:
playerAccessor.ifPresent(p -> p.sendMessage("Hi"));

// Or with default:
Player player = playerAccessor.orElse(defaultPlayer);
```

### Accessor Types

- `Accessor<T>` - Basic accessor
- `LazyAccessor<T>` - Lazy initialization
- `CachedAccessor<T>` - Cached value
- `MutableAccessor<T>` - Mutable value

---

## Configuration System

### What is Configuration?

External settings loaded from files that control mod behavior without code changes.

### Config Types

#### Static Config
Loaded once at startup

```java
@ConfigClass("my-mod")
public class MyConfig {
    @ConfigProperty("feature.enabled")
    private boolean featureEnabled = true;
    
    @ConfigProperty("max.quests")
    private int maxQuests = 10;
}
```

#### Dynamic Config
Reloadable at runtime

```java
Config config = ConfigLoader.load("my-mod.json");
config.watch(); // Auto-reload on file change
```

### Config Files

Location: `config/argonath/<mod-id>.json`

```json
{
  "feature": {
    "enabled": true
  },
  "max": {
    "quests": 10
  }
}
```

---

## Dependency Injection

### What is Dependency Injection?

Automatic provisioning of dependencies instead of manual creation.

### Service Provider

```java
// Register service
ServiceProvider.register(QuestService.class, new QuestServiceImpl());

// Inject automatically
@Inject
private QuestService questService;

// Or get manually
QuestService service = ServiceProvider.get(QuestService.class);
```

### Benefits

- **Testability** - Easy to mock services
- **Flexibility** - Swap implementations
- **Decoupling** - Don't depend on concrete classes

---

## Localization

### What is Localization?

Multi-language support via translation keys.

### Translation Example

```java
// Define key
Text.translatable("quest.complete")
    .with("questName", quest.getName())
    .send(player);
```

**Translation files:**

`lang/en_US.json`:
```json
{
  "quest.complete": "Quest {questName} completed!"
}
```

`lang/es_ES.json`:
```json
{
  "quest.complete": "¡Misión {questName} completada!"
}
```

---

## Performance Concepts

### Lazy Loading

Load data only when needed:

```java
LazyAccessor<ExpensiveData> data = LazyAccessor.of(() -> loadData());
// Data not loaded yet

data.get(); // Loads now
```

### Caching

Store computed results:

```java
CachedAccessor<List<Quest>> quests = CachedAccessor.of(
    () -> database.loadQuests(),
    Duration.ofMinutes(5) // Cache for 5 minutes
);
```

### Async Operations

Non-blocking operations:

```java
CompletableFuture<Quest> futureQuest = 
    QuestLoader.loadAsync("my_quest");

futureQuest.thenAccept(quest -> {
    // Process quest when loaded
});
```

---

## Design Principles

### 1. **Platform Agnostic**
Framework code never directly uses Hytale APIs

### 2. **Composition over Inheritance**
Use components, not deep class hierarchies

### 3. **Immutability**
Build immutable objects when possible

### 4. **Fail-Safe Defaults**
Graceful degradation when things go wrong

### 5. **Clear Separation of Concerns**
Each module has one responsibility

---

## Common Patterns Summary

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Registry** | Centralized lookup | `QuestRegistry.get()` |
| **Builder** | Fluent construction | `QuestBuilder.create()` |
| **Component** | Modular functionality | `entity.addComponent()` |
| **Event** | Decoupled communication | `EventBus.fire()` |
| **Accessor** | Safe object access | `accessor.ifPresent()` |
| **Service** | Dependency injection | `@Inject` |

---

**Next:** [Module Reference](modules.md) | [Dependencies](dependencies.md)
