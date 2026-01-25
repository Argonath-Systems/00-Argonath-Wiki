# Configuration Guide

A comprehensive guide to the Argonath Framework Configuration System.

## Table of Contents

1. [Introduction](#introduction)
2. [Configuration System Overview](#configuration-system-overview)
3. [Creating Configuration Files](#creating-configuration-files)
4. [Loading and Saving Configurations](#loading-and-saving-configurations)
5. [Configuration Types](#configuration-types)
6. [Player-Specific Settings](#player-specific-settings)
7. [Dynamic Configuration](#dynamic-configuration)
8. [Configuration Validation](#configuration-validation)
9. [Hot Reloading](#hot-reloading)
10. [Best Practices](#best-practices)
11. [Migration and Versioning](#migration-and-versioning)
12. [Troubleshooting](#troubleshooting)

---

## Introduction

The Argonath Configuration System provides a robust, flexible framework for managing mod settings, player preferences, and runtime configuration. It supports multiple formats (YAML, JSON, TOML), validation, defaults, and hot reloading.

### Prerequisites

- Understanding of [Core Concepts](../architecture/concepts.md)
- Basic Java knowledge
- Familiarity with configuration file formats

### What You'll Learn

- Configuration architecture
- Creating and managing config files
- Loading and saving configurations
- Validation and type safety
- Player-specific settings
- Dynamic runtime configuration

---

## Configuration System Overview

### Architecture

```
ConfigurationManager
    ├── ConfigLoader (YAML, JSON, TOML)
    ├── ConfigValidator
    ├── ConfigCache
    └── ConfigWatcher (Hot reload)

Configuration
    ├── ConfigSection
    │   ├── Primitive values
    │   ├── Nested sections
    │   └── Collections
    └── ConfigMetadata
```

### Core Components

```java
public interface Configuration {
    // Get values
    <T> T get(String path, T defaultValue);
    String getString(String path);
    int getInt(String path);
    boolean getBoolean(String path);
    List<String> getStringList(String path);
    
    // Set values
    void set(String path, Object value);
    
    // Sections
    ConfigSection getSection(String path);
    Set<String> getKeys();
    
    // Persistence
    void save();
    void reload();
}
```

---

## Creating Configuration Files

### Method 1: YAML Configuration

```yaml
# config.yml
# Main configuration file for MyMod

# General settings
general:
  enabled: true
  debug: false
  language: "en_US"
  auto_save: true
  save_interval: 300  # seconds

# Quest settings
quests:
  enabled: true
  max_active_quests: 10
  allow_quest_sharing: true
  quest_timeout: 3600  # seconds
  
  # Quest rewards
  rewards:
    experience_multiplier: 1.0
    currency_multiplier: 1.0
    bonus_rewards_enabled: true
  
  # Quest display
  display:
    show_tracker: true
    tracker_position: "top_right"
    show_objectives: true
    show_rewards: true

# NPC settings
npcs:
  enabled: true
  spawn_radius: 50
  max_npcs: 100
  respawn_time: 300
  
  # NPC AI
  ai:
    pathfinding_enabled: true
    update_interval: 20  # ticks
    detection_range: 15
    
# UI settings
ui:
  theme: "dark"
  scale: 1.0
  animations_enabled: true
  
  # Window positions (saved automatically)
  windows:
    quest_tracker:
      x: 10
      y: 10
      width: 300
      height: 400
      
# Database settings
database:
  type: "sqlite"
  path: "data/quests.db"
  auto_backup: true
  backup_interval: 86400  # seconds
  
# Performance settings
performance:
  async_saving: true
  cache_size: 100
  cleanup_interval: 600

# Feature flags
features:
  experimental_features: false
  beta_quests: false
  advanced_npcs: true
```

### Method 2: JSON Configuration

```json
{
  "general": {
    "enabled": true,
    "debug": false,
    "language": "en_US"
  },
  "quests": {
    "enabled": true,
    "max_active_quests": 10,
    "rewards": {
      "experience_multiplier": 1.0,
      "currency_multiplier": 1.0
    }
  },
  "npcs": {
    "enabled": true,
    "spawn_radius": 50,
    "ai": {
      "pathfinding_enabled": true,
      "update_interval": 20
    }
  }
}
```

### Method 3: TOML Configuration

```toml
# config.toml

[general]
enabled = true
debug = false
language = "en_US"

[quests]
enabled = true
max_active_quests = 10

[quests.rewards]
experience_multiplier = 1.0
currency_multiplier = 1.0

[npcs]
enabled = true
spawn_radius = 50

[npcs.ai]
pathfinding_enabled = true
update_interval = 20
```

---

## Loading and Saving Configurations

### Basic Configuration Loading

```java
import com.argonath.framework.config.*;

public class ConfigurationExample {
    private Configuration config;
    
    public void loadConfig() {
        // Load from file
        config = ConfigurationManager.load("config.yml");
        
        // With defaults
        config = ConfigurationManager.loadWithDefaults(
            "config.yml",
            getDefaultConfig()
        );
    }
    
    private Configuration getDefaultConfig() {
        Configuration defaults = new YamlConfiguration();
        defaults.set("general.enabled", true);
        defaults.set("general.debug", false);
        defaults.set("quests.max_active_quests", 10);
        return defaults;
    }
    
    public void saveConfig() {
        config.save();
    }
}
```

### Configuration Builder Pattern

```java
public class ConfigurationBuilder {
    public static Configuration createDefaultConfig() {
        return new ConfigurationBuilder()
            .section("general")
                .set("enabled", true)
                .set("debug", false)
                .set("language", "en_US")
                .endSection()
            .section("quests")
                .set("enabled", true)
                .set("max_active_quests", 10)
                .section("rewards")
                    .set("experience_multiplier", 1.0)
                    .set("currency_multiplier", 1.0)
                    .endSection()
                .endSection()
            .build();
    }
}
```

### Type-Safe Configuration

```java
public class TypedConfiguration {
    private final Configuration config;
    
    public TypedConfiguration(Configuration config) {
        this.config = config;
    }
    
    // General settings
    public boolean isEnabled() {
        return config.getBoolean("general.enabled", true);
    }
    
    public void setEnabled(boolean enabled) {
        config.set("general.enabled", enabled);
    }
    
    public String getLanguage() {
        return config.getString("general.language", "en_US");
    }
    
    // Quest settings
    public int getMaxActiveQuests() {
        return config.getInt("quests.max_active_quests", 10);
    }
    
    public double getExperienceMultiplier() {
        return config.getDouble("quests.rewards.experience_multiplier", 1.0);
    }
    
    // Nested configuration
    public QuestConfig getQuestConfig() {
        ConfigSection section = config.getSection("quests");
        return new QuestConfig(section);
    }
}

public class QuestConfig {
    private final ConfigSection section;
    
    public QuestConfig(ConfigSection section) {
        this.section = section;
    }
    
    public boolean isEnabled() {
        return section.getBoolean("enabled", true);
    }
    
    public int getMaxActiveQuests() {
        return section.getInt("max_active_quests", 10);
    }
    
    public RewardConfig getRewardConfig() {
        return new RewardConfig(section.getSection("rewards"));
    }
}
```

---

## Configuration Types

### 1. Global Configuration

Settings that apply to all players.

```java
public class GlobalConfig {
    private static final Configuration config = 
        ConfigurationManager.load("global.yml");
    
    public static boolean isDebugEnabled() {
        return config.getBoolean("debug", false);
    }
    
    public static int getMaxPlayers() {
        return config.getInt("max_players", 100);
    }
    
    public static void setDebugEnabled(boolean debug) {
        config.set("debug", debug);
        config.save();
    }
}
```

### 2. Server Configuration

Server-specific settings.

```java
public class ServerConfig {
    private final Configuration config;
    
    public ServerConfig(String serverId) {
        this.config = ConfigurationManager.load(
            String.format("servers/%s.yml", serverId)
        );
    }
    
    public String getServerName() {
        return config.getString("name", "Unnamed Server");
    }
    
    public int getPort() {
        return config.getInt("port", 25565);
    }
    
    public List<String> getEnabledMods() {
        return config.getStringList("enabled_mods");
    }
}
```

### 3. Feature Configuration

Toggle and configure specific features.

```java
public class FeatureConfig {
    private final Configuration config;
    
    public FeatureConfig() {
        this.config = ConfigurationManager.load("features.yml");
    }
    
    public boolean isFeatureEnabled(String featureName) {
        return config.getBoolean("features." + featureName + ".enabled", false);
    }
    
    public void enableFeature(String featureName) {
        config.set("features." + featureName + ".enabled", true);
        config.save();
    }
    
    public void disableFeature(String featureName) {
        config.set("features." + featureName + ".enabled", false);
        config.save();
    }
    
    public <T> T getFeatureSetting(String featureName, String setting, T defaultValue) {
        return config.get("features." + featureName + "." + setting, defaultValue);
    }
}
```

### 4. Quest Configuration

Quest-specific settings.

```java
public class QuestConfiguration {
    public static Quest loadQuestFromConfig(String questId) {
        Configuration config = ConfigurationManager.load(
            String.format("quests/%s.yml", questId)
        );
        
        return new QuestBuilder(questId)
            .withName(config.getString("name"))
            .withDescription(config.getString("description"))
            .withCategory(config.getString("category"))
            .setRecommendedLevel(config.getInt("recommended_level", 1))
            .withObjectives(loadObjectives(config.getSection("objectives")))
            .withRewards(loadRewards(config.getSection("rewards")))
            .build();
    }
    
    private static List<QuestObjective> loadObjectives(ConfigSection section) {
        List<QuestObjective> objectives = new ArrayList<>();
        
        for (String key : section.getKeys()) {
            ConfigSection objSection = section.getSection(key);
            String type = objSection.getString("type");
            
            QuestObjective objective = switch (type) {
                case "collect" -> createCollectObjective(objSection);
                case "kill" -> createKillObjective(objSection);
                case "talk" -> createTalkObjective(objSection);
                default -> null;
            };
            
            if (objective != null) {
                objectives.add(objective);
            }
        }
        
        return objectives;
    }
    
    private static CollectObjective createCollectObjective(ConfigSection section) {
        return new CollectObjective(
            section.getString("item"),
            section.getInt("amount")
        );
    }
}
```

---

## Player-Specific Settings

### Player Configuration Manager

```java
public class PlayerConfigManager {
    private final Map<UUID, Configuration> playerConfigs = new ConcurrentHashMap<>();
    private final Path configDirectory;
    
    public PlayerConfigManager(Path configDirectory) {
        this.configDirectory = configDirectory;
    }
    
    public Configuration getPlayerConfig(Player player) {
        return playerConfigs.computeIfAbsent(player.getUUID(), uuid -> {
            Path configFile = configDirectory.resolve(uuid + ".yml");
            return ConfigurationManager.loadOrCreate(configFile.toString());
        });
    }
    
    public void savePlayerConfig(Player player) {
        Configuration config = playerConfigs.get(player.getUUID());
        if (config != null) {
            config.save();
        }
    }
    
    public void saveAllPlayerConfigs() {
        playerConfigs.values().forEach(Configuration::save);
    }
    
    public void unloadPlayerConfig(Player player) {
        Configuration config = playerConfigs.remove(player.getUUID());
        if (config != null) {
            config.save();
        }
    }
}
```

### Player Settings

```java
public class PlayerSettings {
    private final Configuration config;
    
    public PlayerSettings(Player player) {
        this.config = PlayerConfigManager.getInstance().getPlayerConfig(player);
    }
    
    // UI Settings
    public boolean isQuestTrackerEnabled() {
        return config.getBoolean("ui.quest_tracker.enabled", true);
    }
    
    public void setQuestTrackerEnabled(boolean enabled) {
        config.set("ui.quest_tracker.enabled", enabled);
        save();
    }
    
    public String getQuestTrackerPosition() {
        return config.getString("ui.quest_tracker.position", "top_right");
    }
    
    // Notification Settings
    public boolean areNotificationsEnabled() {
        return config.getBoolean("notifications.enabled", true);
    }
    
    public NotificationType getNotificationStyle() {
        String style = config.getString("notifications.style", "POPUP");
        return NotificationType.valueOf(style);
    }
    
    // Gameplay Settings
    public boolean isAutoAcceptEnabled() {
        return config.getBoolean("gameplay.auto_accept_quests", false);
    }
    
    public int getMaxTrackedQuests() {
        return config.getInt("gameplay.max_tracked_quests", 5);
    }
    
    // Keybindings
    public String getQuestLogKeybind() {
        return config.getString("keybinds.quest_log", "L");
    }
    
    public void setQuestLogKeybind(String key) {
        config.set("keybinds.quest_log", key);
        save();
    }
    
    private void save() {
        config.save();
    }
}
```

### Player Preferences

```java
public class PlayerPreferences {
    public static class Preference<T> {
        private final String key;
        private final T defaultValue;
        
        public Preference(String key, T defaultValue) {
            this.key = key;
            this.defaultValue = defaultValue;
        }
        
        public T get(Configuration config) {
            return config.get(key, defaultValue);
        }
        
        public void set(Configuration config, T value) {
            config.set(key, value);
        }
    }
    
    // Define preferences
    public static final Preference<Boolean> QUEST_TRACKER_ENABLED = 
        new Preference<>("ui.quest_tracker", true);
    
    public static final Preference<Integer> MAX_TRACKED_QUESTS = 
        new Preference<>("gameplay.max_tracked", 5);
    
    public static final Preference<String> THEME = 
        new Preference<>("ui.theme", "dark");
    
    // Usage
    public static <T> T get(Player player, Preference<T> preference) {
        Configuration config = getPlayerConfig(player);
        return preference.get(config);
    }
    
    public static <T> void set(Player player, Preference<T> preference, T value) {
        Configuration config = getPlayerConfig(player);
        preference.set(config, value);
        config.save();
    }
}
```

---

## Dynamic Configuration

### Runtime Configuration Changes

```java
public class DynamicConfig {
    private final Configuration config;
    private final List<ConfigChangeListener> listeners = new ArrayList<>();
    
    public void set(String path, Object value) {
        Object oldValue = config.get(path, null);
        config.set(path, value);
        
        // Notify listeners
        ConfigChangeEvent event = new ConfigChangeEvent(path, oldValue, value);
        listeners.forEach(listener -> listener.onConfigChange(event));
    }
    
    public void addChangeListener(ConfigChangeListener listener) {
        listeners.add(listener);
    }
    
    public void addChangeListener(String path, ConfigChangeListener listener) {
        listeners.add(event -> {
            if (event.getPath().equals(path)) {
                listener.onConfigChange(event);
            }
        });
    }
}

public interface ConfigChangeListener {
    void onConfigChange(ConfigChangeEvent event);
}

public class ConfigChangeEvent {
    private final String path;
    private final Object oldValue;
    private final Object newValue;
    
    public ConfigChangeEvent(String path, Object oldValue, Object newValue) {
        this.path = path;
        this.oldValue = oldValue;
        this.newValue = newValue;
    }
    
    // Getters...
}
```

### Configuration Watchers

```java
public class ConfigWatcher {
    private final Path configFile;
    private final Configuration config;
    private final WatchService watchService;
    
    public ConfigWatcher(Path configFile) throws IOException {
        this.configFile = configFile;
        this.config = ConfigurationManager.load(configFile.toString());
        this.watchService = FileSystems.getDefault().newWatchService();
        
        configFile.getParent().register(
            watchService,
            StandardWatchEventKinds.ENTRY_MODIFY
        );
        
        startWatching();
    }
    
    private void startWatching() {
        new Thread(() -> {
            while (true) {
                try {
                    WatchKey key = watchService.take();
                    
                    for (WatchEvent<?> event : key.pollEvents()) {
                        if (event.context().toString().equals(configFile.getFileName().toString())) {
                            onConfigFileChanged();
                        }
                    }
                    
                    key.reset();
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();
    }
    
    private void onConfigFileChanged() {
        config.reload();
        System.out.println("Configuration reloaded");
    }
}
```

---

## Configuration Validation

### Schema Validation

```java
public class ConfigValidator {
    public static class ValidationResult {
        private final boolean valid;
        private final List<String> errors;
        
        public ValidationResult(boolean valid, List<String> errors) {
            this.valid = valid;
            this.errors = errors;
        }
        
        public boolean isValid() {
            return valid;
        }
        
        public List<String> getErrors() {
            return errors;
        }
    }
    
    public static ValidationResult validate(Configuration config, ConfigSchema schema) {
        List<String> errors = new ArrayList<>();
        
        for (ConfigField field : schema.getFields()) {
            if (field.isRequired() && !config.contains(field.getPath())) {
                errors.add("Required field missing: " + field.getPath());
            }
            
            if (config.contains(field.getPath())) {
                Object value = config.get(field.getPath(), null);
                
                if (!field.getType().isInstance(value)) {
                    errors.add(String.format(
                        "Invalid type for %s: expected %s, got %s",
                        field.getPath(),
                        field.getType().getSimpleName(),
                        value.getClass().getSimpleName()
                    ));
                }
                
                if (!field.isValid(value)) {
                    errors.add("Invalid value for " + field.getPath() + ": " + value);
                }
            }
        }
        
        return new ValidationResult(errors.isEmpty(), errors);
    }
}

public class ConfigSchema {
    private final List<ConfigField> fields = new ArrayList<>();
    
    public ConfigSchema field(String path, Class<?> type) {
        fields.add(new ConfigField(path, type, false));
        return this;
    }
    
    public ConfigSchema requiredField(String path, Class<?> type) {
        fields.add(new ConfigField(path, type, true));
        return this;
    }
    
    public ConfigSchema fieldWithValidator(String path, Class<?> type, Predicate<Object> validator) {
        ConfigField field = new ConfigField(path, type, false);
        field.setValidator(validator);
        fields.add(field);
        return this;
    }
    
    public List<ConfigField> getFields() {
        return fields;
    }
}

public class ConfigField {
    private final String path;
    private final Class<?> type;
    private final boolean required;
    private Predicate<Object> validator;
    
    public ConfigField(String path, Class<?> type, boolean required) {
        this.path = path;
        this.type = type;
        this.required = required;
    }
    
    public void setValidator(Predicate<Object> validator) {
        this.validator = validator;
    }
    
    public boolean isValid(Object value) {
        return validator == null || validator.test(value);
    }
    
    // Getters...
}
```

### Example Usage

```java
public class ConfigValidationExample {
    public static void validateQuestConfig() {
        Configuration config = ConfigurationManager.load("quest.yml");
        
        ConfigSchema schema = new ConfigSchema()
            .requiredField("name", String.class)
            .requiredField("description", String.class)
            .field("recommended_level", Integer.class)
            .fieldWithValidator("max_active_quests", Integer.class, 
                value -> (int) value > 0 && (int) value <= 20)
            .fieldWithValidator("experience_multiplier", Double.class,
                value -> (double) value >= 0.1 && (double) value <= 10.0);
        
        ValidationResult result = ConfigValidator.validate(config, schema);
        
        if (!result.isValid()) {
            System.err.println("Configuration validation failed:");
            result.getErrors().forEach(System.err::println);
        }
    }
}
```

---

## Hot Reloading

### Hot Reload Manager

```java
public class HotReloadManager {
    private final Map<String, Configuration> configs = new ConcurrentHashMap<>();
    private final Map<String, List<ReloadListener>> listeners = new ConcurrentHashMap<>();
    
    public void registerConfig(String id, Configuration config) {
        configs.put(id, config);
    }
    
    public void addReloadListener(String configId, ReloadListener listener) {
        listeners.computeIfAbsent(configId, k -> new ArrayList<>()).add(listener);
    }
    
    public void reload(String configId) {
        Configuration config = configs.get(configId);
        if (config != null) {
            config.reload();
            notifyListeners(configId, config);
        }
    }
    
    public void reloadAll() {
        configs.forEach((id, config) -> {
            config.reload();
            notifyListeners(id, config);
        });
    }
    
    private void notifyListeners(String configId, Configuration config) {
        List<ReloadListener> configListeners = listeners.get(configId);
        if (configListeners != null) {
            configListeners.forEach(listener -> listener.onReload(config));
        }
    }
}

@FunctionalInterface
public interface ReloadListener {
    void onReload(Configuration config);
}
```

### Reload Command

```java
public class ReloadCommand {
    @Command("config reload")
    @Permission("admin.config.reload")
    public void reload(CommandSender sender, String configId) {
        if (configId.equals("all")) {
            HotReloadManager.getInstance().reloadAll();
            sender.sendMessage("All configurations reloaded");
        } else {
            HotReloadManager.getInstance().reload(configId);
            sender.sendMessage("Configuration reloaded: " + configId);
        }
    }
}
```

---

## Best Practices

### DO:
- ✅ Use meaningful configuration keys
- ✅ Provide sensible defaults
- ✅ Validate configuration values
- ✅ Document configuration options
- ✅ Use type-safe accessors
- ✅ Handle missing/invalid values gracefully
- ✅ Version your configurations

### DON'T:
- ❌ Store sensitive data in plain text
- ❌ Use hard-coded values
- ❌ Ignore validation errors
- ❌ Forget to save changes
- ❌ Mix different concerns in one file
- ❌ Use overly complex nested structures

---

## Migration and Versioning

### Configuration Versioning

```java
public class ConfigMigration {
    public static Configuration migrate(Configuration config) {
        int version = config.getInt("version", 1);
        
        while (version < CURRENT_VERSION) {
            switch (version) {
                case 1 -> migrateV1ToV2(config);
                case 2 -> migrateV2ToV3(config);
                // Add more migrations...
            }
            version++;
        }
        
        config.set("version", CURRENT_VERSION);
        return config;
    }
    
    private static void migrateV1ToV2(Configuration config) {
        // Rename keys
        if (config.contains("old_key")) {
            config.set("new_key", config.get("old_key", null));
            config.remove("old_key");
        }
        
        // Add new defaults
        config.setDefault("new_feature.enabled", true);
    }
}
```

---

## Troubleshooting

### Configuration Not Loading

Check file path, permissions, and format validity.

### Values Not Saving

Verify save() is called and file is writable.

### Type Mismatches

Use proper getters and validate types.

For more information:
- [Storage Guide](storage.md)
- [Quest Development](quest-development.md)
- [API Reference](../api/framework-config.md)
