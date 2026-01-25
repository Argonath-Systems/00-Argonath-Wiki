# Module Reference

Comprehensive reference for all Argonath Systems modules.

## Module Architecture

Argonath Systems is organized into layers, with each module having specific responsibilities and dependencies.

## Layer 1: Platform

Foundation modules that provide core abstractions and SDK.

### platform-core

**Purpose:** Core platform abstractions and interfaces

**Package:** `com.hytale.argonath.platform.core`

**Key Components:**
- `Platform` - Central platform interface
- `PlatformProvider` - Platform discovery service
- `ModuleRegistry` - Module registration system
- `EventBus` - Cross-module event system
- `Logger` - Platform logging

**Dependencies:** None (foundation module)

**When to Use:**
- Always required for any Argonath mod
- Building platform-agnostic systems
- Creating core utilities

**Example:**
```java
import com.hytale.argonath.platform.core.Platform;

Platform platform = Platform.getInstance();
platform.getLogger().info("Hello from platform!");
```

---

### platform-sdk

**Purpose:** Development SDK and utilities for mod developers

**Package:** `com.hytale.argonath.platform.sdk`

**Key Components:**
- `ModInitializer` - Mod entry point interface
- `CommandRegistry` - Command registration API
- `ConfigLoader` - Configuration utilities
- `ResourceManager` - Resource loading
- `ServiceProvider` - Dependency injection

**Dependencies:**
- `platform-core`

**When to Use:**
- Building mods (always needed)
- Registering mod components
- Accessing platform services

**Example:**
```java
public class MyMod implements ModInitializer {
    @Override
    public void onInitialize() {
        // Mod initialization
    }
}
```

---

## Layer 2: Adapters & Frameworks

### adapter-hytale

**Purpose:** Hytale game integration adapter

**Package:** `com.hytale.argonath.adapter.hytale`

**Key Components:**
- `HytaleEventAdapter` - Maps Hytale events to platform events
- `HytalePlayerAdapter` - Player abstraction
- `HytaleWorldAdapter` - World abstraction
- `HytaleEntityAdapter` - Entity abstraction

**Dependencies:**
- `platform-core`
- `platform-sdk`
- Hytale Mod API

**When to Use:**
- Running mods in Hytale
- Accessing Hytale-specific features

---

### adapter-mod-api

**Purpose:** Generic mod API adapter (for other platforms)

**Package:** `com.hytale.argonath.adapter.modapi`

**Key Components:**
- Generic mod API abstractions
- Platform-agnostic event mapping

**Dependencies:**
- `platform-core`
- `platform-sdk`

**When to Use:**
- Porting to other platforms
- Testing without Hytale

---

### framework-core

**Purpose:** Core framework utilities and base classes

**Package:** `com.hytale.argonath.framework.core`

**Key Components:**
- `Builder<T>` - Generic builder pattern base
- `Registry<T>` - Generic registry implementation
- `Component` - Component system
- `Lifecycle` - Component lifecycle management
- `Validator` - Validation utilities

**Dependencies:**
- `platform-core`
- `platform-sdk`

**When to Use:**
- Building custom frameworks
- Using component patterns
- Need validation/registry utilities

**Example:**
```java
public class QuestRegistry extends Registry<Quest> {
    // Inherits registration logic
}
```

---

### framework-accessor

**Purpose:** Safe accessor patterns for game objects

**Package:** `com.hytale.argonath.framework.accessor`

**Key Components:**
- `Accessor<T>` - Safe object access interface
- `LazyAccessor<T>` - Lazy-loaded accessor
- `CachedAccessor<T>` - Cached accessor
- `MutableAccessor<T>` - Mutable accessor

**Dependencies:**
- `framework-core`

**When to Use:**
- Accessing potentially unavailable objects
- Delayed initialization
- Thread-safe access patterns

**Example:**
```java
Accessor<Player> playerAccessor = Accessor.of(() -> getPlayer());
if (playerAccessor.isPresent()) {
    playerAccessor.get().sendMessage("Hello!");
}
```

---

## Layer 3: Utility Frameworks

### framework-config

**Purpose:** Configuration management system

**Package:** `com.hytale.argonath.framework.config`

**Key Components:**
- `Config` - Configuration interface
- `ConfigLoader` - JSON/YAML config loading
- `ConfigNode` - Configuration tree node
- `ConfigSerializer` - Config serialization
- `@ConfigProperty` - Property annotation

**Dependencies:**
- `framework-core`
- `framework-accessor`

**When to Use:**
- Loading mod configuration
- Player-specific settings
- Dynamic configuration

**Example:**
```java
@ConfigClass("my-mod")
public class MyConfig {
    @ConfigProperty("feature.enabled")
    private boolean featureEnabled = true;
}

Config config = ConfigLoader.load(MyConfig.class);
```

---

### framework-storage

**Purpose:** Data persistence and storage

**Package:** `com.hytale.argonath.framework.storage`

**Key Components:**
- `Storage<K, V>` - Key-value storage interface
- `PlayerStorage` - Per-player data storage
- `WorldStorage` - Per-world data storage
- `DatabaseAdapter` - Database integration
- `Serializer<T>` - Object serialization

**Dependencies:**
- `framework-core`
- `framework-config`

**When to Use:**
- Saving player data
- Persistent world state
- Database integration

**Example:**
```java
PlayerStorage<QuestProgress> storage = 
    PlayerStorage.create("quests", QuestProgress.class);

storage.save(player, questProgress);
QuestProgress loaded = storage.load(player);
```

---

### framework-text-styling

**Purpose:** Text formatting and styling

**Package:** `com.hytale.argonath.framework.text`

**Key Components:**
- `Text` - Styled text builder
- `TextColor` - Color utilities
- `TextStyle` - Style formatting
- `TextComponent` - Rich text components
- `PlaceholderResolver` - Dynamic text replacement

**Dependencies:**
- `framework-core`

**When to Use:**
- Formatted chat messages
- UI text
- Localization

**Example:**
```java
Text.builder()
    .append("Warning: ", TextColor.RED, TextStyle.BOLD)
    .append("Low health!", TextColor.YELLOW)
    .build()
    .send(player);
```

---

## Layer 4: Feature Frameworks

### framework-condition

**Purpose:** Condition evaluation system

**Package:** `com.hytale.argonath.framework.condition`

**Key Components:**
- `Condition` - Condition interface
- `ConditionBuilder` - Fluent condition creation
- `ConditionEvaluator` - Condition evaluation engine
- `CompositeCondition` - AND/OR/NOT logic
- Built-in conditions (level, location, item, quest, time)

**Dependencies:**
- `framework-core`
- `framework-accessor`

**When to Use:**
- Quest requirements
- NPC dialog conditions
- Feature gating

**Example:**
```java
Condition condition = Condition.and(
    Condition.level(5),
    Condition.hasItem("gold_coin", 100),
    Condition.questCompleted("tutorial")
);

if (condition.test(player)) {
    // Allow action
}
```

---

### framework-objective

**Purpose:** Objective tracking system

**Package:** `com.hytale.argonath.framework.objective`

**Key Components:**
- `Objective` - Objective interface
- `ObjectiveType` - Built-in objective types
- `ObjectiveProgress` - Progress tracking
- `ObjectiveBuilder` - Fluent creation
- Types: KILL, COLLECT, TALK, REACH, CRAFT, etc.

**Dependencies:**
- `framework-core`
- `framework-condition`
- `framework-storage`

**When to Use:**
- Quest objectives
- Achievement tracking
- Progress systems

**Example:**
```java
Objective killGoblins = Objective.builder()
    .type(ObjectiveType.KILL_ENTITY)
    .target("goblin")
    .count(10)
    .description("Kill 10 goblins")
    .build();
```

---

### framework-npc

**Purpose:** NPC management and dialog system

**Package:** `com.hytale.argonath.framework.npc`

**Key Components:**
- `NPC` - NPC entity
- `NPCBuilder` - Fluent NPC creation
- `DialogNode` - Dialog tree node
- `DialogChoice` - Player dialog choices
- `NPCRegistry` - NPC registration
- `InteractionHandler` - NPC interaction logic

**Dependencies:**
- `framework-core`
- `framework-condition`
- `framework-text-styling`

**When to Use:**
- Quest NPCs
- Merchants
- Interactive characters
- Dialog systems

**Example:**
```java
NPC elder = NPCBuilder.create("village_elder")
    .name("Elder Marcus")
    .model("elder")
    .addDialog(
        DialogNode.builder()
            .text("Welcome, traveler!")
            .option("Hello!", "greeting")
            .build()
    )
    .build();
```

---

### framework-quest

**Purpose:** Complete quest system

**Package:** `com.hytale.argonath.framework.quest`

**Key Components:**
- `Quest` - Quest definition
- `QuestBuilder` - Fluent quest creation
- `QuestRegistry` - Quest registration
- `QuestProgress` - Player quest progress
- `QuestManager` - Quest lifecycle
- `QuestReward` - Reward system
- `QuestChain` - Quest dependencies

**Dependencies:**
- `framework-core`
- `framework-condition`
- `framework-objective`
- `framework-npc`
- `framework-storage`

**When to Use:**
- Implementing quests
- Quest chains
- Reward systems

**Example:**
```java
Quest quest = QuestBuilder.create("my_quest")
    .name("Epic Adventure")
    .description("A grand quest!")
    .addObjective(killGoblins)
    .addReward(reward -> reward.gold(100))
    .build();
```

---

### framework-ui

**Purpose:** UI framework and components

**Package:** `com.hytale.argonath.framework.ui`

**Key Components:**
- `UIComponent` - Base UI component
- `Window` - Window/panel container
- `Button`, `Label`, `TextField` - UI widgets
- `Layout` - Layout managers
- `UIRegistry` - UI registration
- `EventHandler` - UI event handling

**Dependencies:**
- `framework-core`
- `framework-text-styling`
- `adapter-hytale`

**When to Use:**
- Custom UIs
- Quest tracker UI
- Inventory GUIs
- HUDs

**Example:**
```java
Window questWindow = Window.builder()
    .title("Active Quests")
    .size(400, 300)
    .addChild(Label.of("My Quest"))
    .addChild(Button.of("Close", this::close))
    .build();
```

---

## Layer 5 & 6: Mods

### mod-quest-tracker

**Purpose:** Visual quest tracking mod

**Package:** `com.hytale.argonath.mod.questtracker`

**Key Components:**
- Quest list UI
- Quest progress HUD
- Quest waypoints
- Quest notifications

**Dependencies:**
- `framework-quest`
- `framework-ui`
- All dependencies of quest framework

**When to Use:**
- Displaying active quests
- Quest UI for players
- In-game quest tracking

---

## Dependency Graph

```
┌─────────────────┐
│  platform-core  │ (Foundation)
└────────┬────────┘
         │
┌────────▼────────┐
│  platform-sdk   │
└────────┬────────┘
         │
    ┌────┴────────────────────────┐
    │                             │
┌───▼──────────┐        ┌─────────▼──────────┐
│adapter-hytale│        │ framework-core      │
└──────────────┘        └─────────┬───────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
            ┌───────▼─────┐  ┌────▼────┐  ┌────▼────────┐
            │framework-   │  │framework│  │framework-   │
            │accessor     │  │-config  │  │text-styling │
            └─────────────┘  └────┬────┘  └─────────────┘
                                  │
                        ┌─────────▼──────────┐
                        │framework-storage   │
                        └────────────────────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
            ┌───────▼────────┐ ┌─▼────────┐ ┌──▼───────────┐
            │framework-      │ │framework-│ │framework-npc │
            │condition       │ │objective │ └──────────────┘
            └────────────────┘ └──────────┘
                    │             │
                    └──────┬──────┘
                           │
                   ┌───────▼──────────┐
                   │framework-quest   │
                   └──────────────────┘
                           │
                   ┌───────▼──────────┐
                   │mod-quest-tracker │
                   └──────────────────┘
```

## Module Selection Guide

### For Quest Mods
```
Required:
- platform-core + platform-sdk
- adapter-hytale
- framework-core
- framework-quest (includes all quest dependencies)

Optional:
- framework-ui (for custom UIs)
- mod-quest-tracker (visual tracking)
```

### For UI Mods
```
Required:
- platform-core + platform-sdk
- adapter-hytale
- framework-core
- framework-ui

Optional:
- framework-text-styling (rich text)
```

### For Data Mods
```
Required:
- platform-core + platform-sdk
- adapter-hytale
- framework-core
- framework-storage

Optional:
- framework-config (configuration)
```

### For Library Development
```
Minimal:
- platform-core (compileOnly)
- platform-sdk (compileOnly)

Add as needed:
- framework-core (common utilities)
```

## Version Compatibility

All modules within the same major version are compatible:

| Version | Compatible | Notes |
|---------|------------|-------|
| 1.0.x | ✅ All 1.0.x | Patch updates |
| 1.x.0 | ✅ All 1.x | Minor features |
| 2.x | ❌ With 1.x | Breaking changes |

## Module Status

| Module | Status | Stability |
|--------|--------|-----------|
| platform-core | ✅ Stable | Production |
| platform-sdk | ✅ Stable | Production |
| adapter-hytale | ✅ Stable | Production |
| framework-core | ✅ Stable | Production |
| framework-quest | ✅ Stable | Production |
| framework-ui | 🚧 Beta | Testing |
| All others | ✅ Stable | Production |

---

**Next:** [Core Concepts](concepts.md) | [Architecture Overview](overview.md)
