# Platform Core API Reference

**Package:** `com.argonathsystems.platform.core`  
**Version:** 1.0.0  
**Since:** 1.0.0

## Overview

The Platform Core module provides the foundational infrastructure for the Argonath Systems framework. It implements platform-agnostic abstractions for dependency injection, event handling, module management, and logging.

## Table of Contents

- [Platform](#platform)
- [PlatformProvider](#platformprovider)
- [EventBus](#eventbus)
- [Logger](#logger)
- [ModuleRegistry](#moduleregistry)
- [Module Interface](#module-interface)
- [ServiceRegistry](#serviceregistry)
- [Configuration](#configuration)
- [Lifecycle Management](#lifecycle-management)

---

## Platform

The central entry point for accessing platform services and modules.

### Class Declaration

```java
public interface Platform {
    String getName();
    String getVersion();
    PlatformProvider getProvider();
    EventBus getEventBus();
    Logger getLogger(String name);
    Logger getLogger(Class<?> clazz);
    ModuleRegistry getModuleRegistry();
    ServiceRegistry getServiceRegistry();
    ConfigurationManager getConfigurationManager();
    boolean isEnabled();
    void shutdown();
}
```

### Methods

#### `getName()`

Returns the platform implementation name.

```java
String platformName = platform.getName();
// Returns: "Hytale" or "Spigot" or "Mock" depending on adapter
```

**Returns:** Platform implementation name  
**Thread Safety:** Thread-safe

---

#### `getVersion()`

Returns the platform version string.

```java
String version = platform.getVersion();
// Returns: "1.0.0"
```

**Returns:** Platform version  
**Thread Safety:** Thread-safe

---

#### `getProvider()`

Returns the platform provider for accessing platform-specific services.

```java
PlatformProvider provider = platform.getProvider();
Player player = provider.getPlayer(uuid);
```

**Returns:** `PlatformProvider` instance  
**Thread Safety:** Thread-safe

---

#### `getEventBus()`

Returns the global event bus for event publication and subscription.

```java
EventBus eventBus = platform.getEventBus();
eventBus.post(new QuestCompletedEvent(player, quest));
```

**Returns:** `EventBus` instance  
**Thread Safety:** Thread-safe  
**See Also:** [EventBus](#eventbus)

---

#### `getLogger(String name)`

Creates a named logger instance.

```java
Logger logger = platform.getLogger("QuestSystem");
logger.info("Quest system initialized");
```

**Parameters:**
- `name` - Logger name (typically component or class name)

**Returns:** `Logger` instance  
**Thread Safety:** Thread-safe  
**See Also:** [Logger](#logger)

---

#### `getLogger(Class<?> clazz)`

Creates a logger instance named after a class.

```java
Logger logger = platform.getLogger(QuestManager.class);
logger.debug("Loading quests from storage");
```

**Parameters:**
- `clazz` - Class to derive logger name from

**Returns:** `Logger` instance  
**Thread Safety:** Thread-safe

---

#### `getModuleRegistry()`

Returns the module registry for registering and querying modules.

```java
ModuleRegistry registry = platform.getModuleRegistry();
QuestModule questModule = registry.getModule("quest", QuestModule.class);
```

**Returns:** `ModuleRegistry` instance  
**Thread Safety:** Thread-safe  
**See Also:** [ModuleRegistry](#moduleregistry)

---

#### `getServiceRegistry()`

Returns the service registry for dependency injection.

```java
ServiceRegistry services = platform.getServiceRegistry();
services.register(QuestManager.class, new QuestManagerImpl());
```

**Returns:** `ServiceRegistry` instance  
**Thread Safety:** Thread-safe  
**See Also:** [ServiceRegistry](#serviceregistry)

---

#### `shutdown()`

Gracefully shuts down the platform and all registered modules.

```java
platform.shutdown();
// All modules are disabled in reverse dependency order
// All resources are released
// Event listeners are unregistered
```

**Thread Safety:** Thread-safe (idempotent)  
**Side Effects:** Disables all modules, releases resources

---

### Example Usage

```java
public class QuestPlugin {
    private final Platform platform;
    private final Logger logger;
    
    public QuestPlugin(Platform platform) {
        this.platform = platform;
        this.logger = platform.getLogger(QuestPlugin.class);
    }
    
    public void onEnable() {
        logger.info("Initializing quest system on platform: {}", 
                    platform.getName());
        
        // Register module
        ModuleRegistry registry = platform.getModuleRegistry();
        registry.registerModule(new QuestModule());
        
        // Register services
        ServiceRegistry services = platform.getServiceRegistry();
        services.register(QuestManager.class, new QuestManagerImpl());
        
        // Subscribe to events
        EventBus eventBus = platform.getEventBus();
        eventBus.subscribe(QuestCompletedEvent.class, this::onQuestCompleted);
        
        logger.info("Quest system initialized successfully");
    }
    
    private void onQuestCompleted(QuestCompletedEvent event) {
        logger.info("Quest completed: {} by {}", 
                    event.getQuest().getId(), 
                    event.getPlayer().getName());
    }
}
```

---

## PlatformProvider

Provides access to platform-specific implementations and game entities.

### Class Declaration

```java
public interface PlatformProvider {
    Player getPlayer(UUID uuid);
    Player getPlayer(String name);
    Collection<Player> getOnlinePlayers();
    World getWorld(String name);
    Collection<World> getWorlds();
    Server getServer();
    Scheduler getScheduler();
    <T> T getAdapter(Class<T> adapterClass);
}
```

### Methods

#### `getPlayer(UUID uuid)`

Retrieves a player by unique identifier.

```java
PlatformProvider provider = platform.getProvider();
Player player = provider.getPlayer(uuid);
if (player != null && player.isOnline()) {
    player.sendMessage("Quest available!");
}
```

**Parameters:**
- `uuid` - Player's unique identifier

**Returns:** `Player` instance or `null` if not found  
**Thread Safety:** Thread-safe

---

#### `getPlayer(String name)`

Retrieves a player by username.

```java
Player player = provider.getPlayer("Steve");
if (player != null) {
    questManager.assignQuest(player, "main_quest_1");
}
```

**Parameters:**
- `name` - Player's username (case-insensitive)

**Returns:** `Player` instance or `null` if not found  
**Thread Safety:** Thread-safe

---

#### `getOnlinePlayers()`

Returns all currently online players.

```java
Collection<Player> players = provider.getOnlinePlayers();
for (Player player : players) {
    if (questManager.hasActiveQuest(player)) {
        player.sendMessage("You have active quests!");
    }
}
```

**Returns:** Unmodifiable collection of online players  
**Thread Safety:** Thread-safe  
**Note:** Returns a snapshot; collection is not live-updating

---

#### `getWorld(String name)`

Retrieves a world by name.

```java
World world = provider.getWorld("overworld");
Location spawnLocation = world.getSpawnLocation();
```

**Parameters:**
- `name` - World name

**Returns:** `World` instance or `null` if not found  
**Thread Safety:** Thread-safe

---

#### `getScheduler()`

Returns the task scheduler for async and delayed execution.

```java
Scheduler scheduler = provider.getScheduler();
scheduler.runTaskLater(() -> {
    player.sendMessage("Quest reminder!");
}, 20 * 60); // 60 seconds later
```

**Returns:** `Scheduler` instance  
**Thread Safety:** Thread-safe

---

### Example Usage

```java
public class QuestRewardService {
    private final PlatformProvider provider;
    
    public void giveReward(UUID playerId, QuestReward reward) {
        Player player = provider.getPlayer(playerId);
        if (player == null) {
            logger.warn("Cannot give reward: Player {} not online", playerId);
            return;
        }
        
        // Execute reward on main thread
        Scheduler scheduler = provider.getScheduler();
        scheduler.runTask(() -> {
            player.getInventory().addItem(reward.getItems());
            player.sendMessage("Quest reward received!");
        });
    }
}
```

---

## EventBus

High-performance event publication and subscription system.

### Class Declaration

```java
public interface EventBus {
    <T extends Event> void post(T event);
    <T extends Event> CompletableFuture<T> postAsync(T event);
    <T extends Event> void subscribe(Class<T> eventType, EventListener<T> listener);
    <T extends Event> void subscribe(Class<T> eventType, EventPriority priority, EventListener<T> listener);
    <T extends Event> void unsubscribe(Class<T> eventType, EventListener<T> listener);
    void unsubscribeAll(Object listener);
    int getSubscriberCount(Class<? extends Event> eventType);
}
```

### Methods

#### `post(T event)`

Publishes an event synchronously to all registered listeners.

```java
EventBus eventBus = platform.getEventBus();
QuestCompletedEvent event = new QuestCompletedEvent(player, quest);
eventBus.post(event);

if (event.isCancelled()) {
    logger.info("Quest completion was cancelled");
}
```

**Parameters:**
- `event` - Event instance to publish

**Thread Safety:** Thread-safe  
**Execution:** Synchronous, listeners executed in priority order  
**Exception Handling:** Listener exceptions are logged but don't stop propagation

---

#### `postAsync(T event)`

Publishes an event asynchronously.

```java
CompletableFuture<QuestCompletedEvent> future = 
    eventBus.postAsync(new QuestCompletedEvent(player, quest));

future.thenAccept(event -> {
    logger.info("Event processing complete. Cancelled: {}", event.isCancelled());
});
```

**Parameters:**
- `event` - Event instance to publish

**Returns:** `CompletableFuture<T>` completed after all listeners execute  
**Thread Safety:** Thread-safe  
**Execution:** Asynchronous on event executor thread pool

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

**Thread Safety:** Thread-safe

---

#### `subscribe(Class<T> eventType, EventPriority priority, EventListener<T> listener)`

Subscribes to events with specific priority.

```java
// High priority listener runs first
eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGH, event -> {
    if (!player.hasPermission("quests.complete")) {
        event.setCancelled(true);
    }
});
```

**Parameters:**
- `eventType` - Event class to listen for
- `priority` - Execution priority (LOWEST, LOW, NORMAL, HIGH, HIGHEST, MONITOR)
- `listener` - Functional listener interface

**Thread Safety:** Thread-safe  
**Note:** MONITOR priority should only observe, not modify events

---

#### `unsubscribe(Class<T> eventType, EventListener<T> listener)`

Removes a specific event listener.

```java
EventListener<QuestCompletedEvent> listener = event -> {
    // Handle event
};

eventBus.subscribe(QuestCompletedEvent.class, listener);
// Later...
eventBus.unsubscribe(QuestCompletedEvent.class, listener);
```

**Parameters:**
- `eventType` - Event class
- `listener` - Listener to remove

**Thread Safety:** Thread-safe

---

### Event Priority

```java
public enum EventPriority {
    LOWEST,   // Runs first, can cancel events early
    LOW,
    NORMAL,   // Default priority
    HIGH,
    HIGHEST,  // Runs last, final modifications
    MONITOR   // Observer only, should not modify events
}
```

### Example Usage

```java
public class QuestEventHandler {
    private final Platform platform;
    private final QuestManager questManager;
    
    public void register() {
        EventBus eventBus = platform.getEventBus();
        
        // Validate quest completion (high priority)
        eventBus.subscribe(QuestCompletedEvent.class, EventPriority.HIGH, 
            this::validateCompletion);
        
        // Process rewards (normal priority)
        eventBus.subscribe(QuestCompletedEvent.class, 
            this::giveRewards);
        
        // Log completion (monitor priority)
        eventBus.subscribe(QuestCompletedEvent.class, EventPriority.MONITOR, 
            this::logCompletion);
    }
    
    private void validateCompletion(QuestCompletedEvent event) {
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        
        if (!questManager.meetsRequirements(player, quest)) {
            event.setCancelled(true);
            player.sendMessage("Quest requirements not met!");
        }
    }
    
    private void giveRewards(QuestCompletedEvent event) {
        if (event.isCancelled()) return;
        
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        
        for (QuestReward reward : quest.getRewards()) {
            reward.apply(player);
        }
    }
    
    private void logCompletion(QuestCompletedEvent event) {
        if (event.isCancelled()) {
            logger.info("Quest completion cancelled: {}", event.getQuest().getId());
        } else {
            logger.info("Quest completed: {} by {}", 
                       event.getQuest().getId(), 
                       event.getPlayer().getName());
        }
    }
}
```

---

## Logger

Structured logging interface with level-based filtering.

### Class Declaration

```java
public interface Logger {
    void trace(String message);
    void trace(String message, Object... args);
    void debug(String message);
    void debug(String message, Object... args);
    void info(String message);
    void info(String message, Object... args);
    void warn(String message);
    void warn(String message, Object... args);
    void warn(String message, Throwable throwable);
    void error(String message);
    void error(String message, Object... args);
    void error(String message, Throwable throwable);
    boolean isTraceEnabled();
    boolean isDebugEnabled();
}
```

### Methods

#### `info(String message, Object... args)`

Logs informational message with SLF4J-style placeholders.

```java
Logger logger = platform.getLogger("QuestSystem");
logger.info("Player {} started quest {}", player.getName(), quest.getId());
```

**Parameters:**
- `message` - Log message with `{}` placeholders
- `args` - Arguments to substitute into placeholders

**Log Level:** INFO  
**Thread Safety:** Thread-safe

---

#### `warn(String message, Throwable throwable)`

Logs warning with exception stack trace.

```java
try {
    questManager.loadQuests();
} catch (IOException e) {
    logger.warn("Failed to load quests from disk", e);
}
```

**Parameters:**
- `message` - Warning message
- `throwable` - Exception to log

**Log Level:** WARN  
**Thread Safety:** Thread-safe

---

#### `error(String message, Throwable throwable)`

Logs error with exception stack trace.

```java
try {
    questManager.saveProgress(player);
} catch (Exception e) {
    logger.error("Critical error saving quest progress for player {}", 
                 player.getName(), e);
}
```

**Parameters:**
- `message` - Error message
- `throwable` - Exception to log

**Log Level:** ERROR  
**Thread Safety:** Thread-safe

---

### Log Levels

| Level | Purpose | Example |
|-------|---------|---------|
| TRACE | Detailed diagnostic information | Method entry/exit, variable values |
| DEBUG | Development debugging | Quest state changes, calculations |
| INFO | General informational messages | Plugin enabled, quest completed |
| WARN | Potentially harmful situations | Config missing optional value |
| ERROR | Error events | Quest save failed, NPC spawn error |

### Example Usage

```java
public class QuestLoader {
    private final Logger logger;
    
    public QuestLoader(Platform platform) {
        this.logger = platform.getLogger(QuestLoader.class);
    }
    
    public List<Quest> loadQuests(Path questDir) {
        logger.info("Loading quests from directory: {}", questDir);
        
        if (!Files.exists(questDir)) {
            logger.warn("Quest directory does not exist: {}", questDir);
            return Collections.emptyList();
        }
        
        List<Quest> quests = new ArrayList<>();
        try (Stream<Path> paths = Files.walk(questDir)) {
            paths.filter(p -> p.toString().endsWith(".yml"))
                 .forEach(path -> {
                     if (logger.isDebugEnabled()) {
                         logger.debug("Loading quest file: {}", path.getFileName());
                     }
                     try {
                         Quest quest = parseQuestFile(path);
                         quests.add(quest);
                         logger.trace("Loaded quest: {}", quest.getId());
                     } catch (Exception e) {
                         logger.error("Failed to load quest from file: {}", path, e);
                     }
                 });
        } catch (IOException e) {
            logger.error("Failed to walk quest directory", e);
        }
        
        logger.info("Loaded {} quests successfully", quests.size());
        return quests;
    }
}
```

---

## ModuleRegistry

Central registry for managing framework modules and their dependencies.

### Class Declaration

```java
public interface ModuleRegistry {
    void registerModule(Module module);
    void enableModule(String moduleId);
    void disableModule(String moduleId);
    <T extends Module> T getModule(String moduleId, Class<T> moduleClass);
    Collection<Module> getAllModules();
    Collection<Module> getEnabledModules();
    boolean isModuleEnabled(String moduleId);
}
```

### Methods

#### `registerModule(Module module)`

Registers a module with the registry.

```java
ModuleRegistry registry = platform.getModuleRegistry();
QuestModule questModule = new QuestModule();
registry.registerModule(questModule);
```

**Parameters:**
- `module` - Module instance to register

**Throws:** `ModuleException` if module ID conflicts or dependencies missing  
**Thread Safety:** Thread-safe

---

#### `enableModule(String moduleId)`

Enables a module and its dependencies in dependency order.

```java
registry.enableModule("quest");
// Dependencies are enabled first: core -> config -> quest
```

**Parameters:**
- `moduleId` - Module identifier

**Throws:** `ModuleException` if module not found or dependencies missing  
**Thread Safety:** Thread-safe  
**Side Effects:** Calls `Module.onEnable()` for module and dependencies

---

#### `getModule(String moduleId, Class<T> moduleClass)`

Retrieves a registered module by ID and type.

```java
QuestModule questModule = registry.getModule("quest", QuestModule.class);
QuestManager manager = questModule.getQuestManager();
```

**Parameters:**
- `moduleId` - Module identifier
- `moduleClass` - Expected module class

**Returns:** Module instance or `null` if not found  
**Throws:** `ClassCastException` if module is wrong type  
**Thread Safety:** Thread-safe

---

### Example Usage

```java
public class FrameworkBootstrap {
    private final Platform platform;
    
    public void initializeFramework() {
        ModuleRegistry registry = platform.getModuleRegistry();
        
        // Register modules in any order (dependencies resolved automatically)
        registry.registerModule(new CoreModule());
        registry.registerModule(new ConfigModule());
        registry.registerModule(new StorageModule());
        registry.registerModule(new NPCModule());
        registry.registerModule(new ObjectiveModule());
        registry.registerModule(new QuestModule());
        
        // Enable modules (dependencies enabled automatically)
        registry.enableModule("quest");
        
        // All dependencies are enabled in correct order:
        // 1. CoreModule
        // 2. ConfigModule, StorageModule (parallel)
        // 3. NPCModule, ObjectiveModule (parallel)
        // 4. QuestModule
    }
    
    public void shutdown() {
        ModuleRegistry registry = platform.getModuleRegistry();
        
        // Disable in reverse dependency order
        registry.disableModule("quest");
    }
}
```

---

## Module Interface

Base interface for all framework modules.

### Class Declaration

```java
public interface Module {
    String getId();
    String getName();
    String getVersion();
    List<String> getDependencies();
    void onEnable(Platform platform);
    void onDisable();
    boolean isEnabled();
}
```

### Methods

#### `getId()`

Returns unique module identifier.

```java
@Override
public String getId() {
    return "quest";
}
```

**Returns:** Unique module ID (lowercase, alphanumeric with hyphens)

---

#### `getDependencies()`

Returns list of module IDs this module depends on.

```java
@Override
public List<String> getDependencies() {
    return Arrays.asList("core", "config", "storage", "objective");
}
```

**Returns:** List of dependency module IDs  
**Note:** Dependencies are enabled before this module

---

#### `onEnable(Platform platform)`

Called when module is enabled.

```java
@Override
public void onEnable(Platform platform) {
    this.platform = platform;
    this.logger = platform.getLogger(QuestModule.class);
    this.questManager = new QuestManagerImpl(platform);
    
    // Register services
    platform.getServiceRegistry()
            .register(QuestManager.class, questManager);
    
    logger.info("Quest module enabled");
}
```

**Parameters:**
- `platform` - Platform instance

**Thread Safety:** Called on main thread  
**Contract:** Must be idempotent

---

### Example Implementation

```java
public class QuestModule implements Module {
    private Platform platform;
    private Logger logger;
    private QuestManager questManager;
    private boolean enabled;
    
    @Override
    public String getId() {
        return "quest";
    }
    
    @Override
    public String getName() {
        return "Quest Framework";
    }
    
    @Override
    public String getVersion() {
        return "1.0.0";
    }
    
    @Override
    public List<String> getDependencies() {
        return Arrays.asList("core", "config", "storage", "objective");
    }
    
    @Override
    public void onEnable(Platform platform) {
        this.platform = platform;
        this.logger = platform.getLogger(QuestModule.class);
        
        logger.info("Enabling Quest Framework v{}", getVersion());
        
        // Initialize managers
        this.questManager = new QuestManagerImpl(platform);
        
        // Register services
        ServiceRegistry services = platform.getServiceRegistry();
        services.register(QuestManager.class, questManager);
        
        // Subscribe to events
        EventBus eventBus = platform.getEventBus();
        eventBus.subscribe(PlayerJoinEvent.class, this::onPlayerJoin);
        
        this.enabled = true;
        logger.info("Quest Framework enabled successfully");
    }
    
    @Override
    public void onDisable() {
        logger.info("Disabling Quest Framework");
        
        // Save all quest progress
        questManager.saveAll();
        
        // Unregister services
        platform.getServiceRegistry()
                .unregister(QuestManager.class);
        
        // Unsubscribe from events
        platform.getEventBus()
                .unsubscribeAll(this);
        
        this.enabled = false;
        logger.info("Quest Framework disabled");
    }
    
    @Override
    public boolean isEnabled() {
        return enabled;
    }
    
    public QuestManager getQuestManager() {
        if (!enabled) {
            throw new IllegalStateException("Quest module is not enabled");
        }
        return questManager;
    }
}
```

---

## ServiceRegistry

Dependency injection container for framework services.

### Class Declaration

```java
public interface ServiceRegistry {
    <T> void register(Class<T> serviceClass, T implementation);
    <T> void register(Class<T> serviceClass, T implementation, String qualifier);
    <T> T getService(Class<T> serviceClass);
    <T> T getService(Class<T> serviceClass, String qualifier);
    <T> void unregister(Class<T> serviceClass);
    boolean hasService(Class<T> serviceClass);
}
```

### Example Usage

```java
// Register services
ServiceRegistry services = platform.getServiceRegistry();
services.register(QuestManager.class, new QuestManagerImpl());
services.register(NPCManager.class, new NPCManagerImpl());
services.register(Database.class, new MySQLDatabase(), "mysql");
services.register(Database.class, new SQLiteDatabase(), "sqlite");

// Retrieve services
QuestManager questManager = services.getService(QuestManager.class);
Database mysql = services.getService(Database.class, "mysql");
```

---

## Configuration

Configuration management interface.

### Example Usage

```java
ConfigurationManager config = platform.getConfigurationManager();

// Load configuration
Configuration questConfig = config.load("quests.yml");

// Read values
String questPrefix = questConfig.getString("prefix", "[Quest]");
int maxActiveQuests = questConfig.getInt("max-active-quests", 5);
boolean debugMode = questConfig.getBoolean("debug", false);

// Nested values
List<String> enabledQuests = questConfig.getStringList("enabled-quests");
```

---

## See Also

- [Event System API](events.md) - Complete event reference
- [Quest Framework API](framework-quest.md) - Quest module API
- [NPC Framework API](framework-npc.md) - NPC module API
- [UI Framework API](framework-ui.md) - UI module API
