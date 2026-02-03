# Breaking Changes Reference

## Introduction

This document provides a comprehensive reference of all breaking changes across Argonath Systems versions. Understanding these changes is critical for successful upgrades and migrations.

**Definition**: A *breaking change* is any modification that requires code changes in dependent projects or results in different runtime behavior for existing functionality.

## Table of Contents

1. [Version 0.x to 1.0](#version-0x-to-10)
2. [Version 1.0 to 2.0](#version-10-to-20)
3. [API Renames and Removals](#api-renames-and-removals)
4. [Configuration Format Changes](#configuration-format-changes)
5. [Database Schema Migrations](#database-schema-migrations)
6. [Migration Scripts and Examples](#migration-scripts-and-examples)
7. [Compatibility Shims](#compatibility-shims)

---

## Version 0.x to 1.0

### Overview

Version 1.0 was the first stable release of Argonath Systems, representing a complete rewrite from the experimental 0.x versions. **Nearly everything changed** between 0.x and 1.0.

### Timeline

- **0.x (Alpha)**: 2023-01-01 to 2023-12-31
- **1.0.0 (Stable)**: 2024-01-15
- **Migration Support**: Ended 2024-07-15

### Major Architectural Changes

#### 1. Module Structure Reorganized

**0.x Structure**:
```
argonath-framework/
├── core/
├── utils/
└── features/
```

**1.0 Structure**:
```
platform-core/
platform-sdk/
framework-core/
framework-*/ (modular components)
```

**Migration Impact**: All import statements need updating.

```java
// 0.x imports
import com.argonath.framework.core.Quest;
import com.argonath.framework.features.QuestManager;

// 1.0 imports
import systems.argonath.quest.Quest;
import systems.argonath.quest.QuestManager;
```

#### 2. Package Naming Convention Changed

**Breaking Change**: Root package changed from `com.argonath` to `systems.argonath`.

**Reason**: Better organization and namespace management.

**Migration**:
```bash
# Automated package rename
find src/ -type f -name "*.java" -exec sed -i 's/com\.argonath/systems.argonath/g' {} +
```

#### 3. Quest System Completely Redesigned

**0.x Quest**:
```java
// Old API (0.x)
Quest quest = new Quest("quest_id", "Quest Name");
quest.addTask(new Task("task1", "Do something"));
quest.setReward(new Reward(ItemStack.of(GOLD), 100));

// Start quest
questManager.assignQuest(player, quest);

// Progress
task.markComplete(player);
```

**1.0 Quest**:
```java
// New API (1.0)
Quest quest = Quest.builder()
    .id("quest_id")
    .displayName(Component.text("Quest Name"))
    .addObjective(Objective.builder()
        .id("obj1")
        .type(ObjectiveType.CUSTOM)
        .description(Component.text("Do something"))
        .build())
    .rewards(RewardSet.builder()
        .addItem(ItemReward.of(Items.GOLD_INGOT, 100))
        .build())
    .build();

// Start quest (different API)
questService.startQuest(player.getUniqueId(), quest.getId());

// Progress (event-driven)
objectiveService.progress(player.getUniqueId(), "obj1", 1);
```

**Migration Complexity**: **HIGH** - Manual rewrite required.

#### 4. Configuration Format Changed

**0.x Configuration** (properties):
```properties
# config.properties
quest.enabled=true
quest.max-active=10
quest.reward-multiplier=1.5
```

**1.0 Configuration** (YAML):
```yaml
# argonath.yml
argonath:
  quest:
    enabled: true
    max-active-per-player: 10
    reward-multiplier: 1.5
```

**Migration Tool**:
```bash
./tools/convert-config.sh --from config.properties --to argonath.yml
```

#### 5. Database Schema Incompatible

**0.x Tables**:
- `quests`
- `player_quests`
- `quest_progress`

**1.0 Tables**:
- `quest_definitions`
- `quest_instances`
- `quest_objectives`
- `player_quest_data`
- `quest_events`

**Migration Required**: Full data migration with downtime.

```sql
-- 0.x to 1.0 data migration (simplified)
-- WARNING: Backup before running!

-- Migrate quest definitions
INSERT INTO quest_definitions (id, definition_data, version)
SELECT quest_id, 
       JSON_OBJECT('name', quest_name, 'description', quest_desc),
       1
FROM quests;

-- Migrate player progress
INSERT INTO player_quest_data (player_id, quest_id, state, started_at)
SELECT player_uuid, quest_id, 
       CASE status 
           WHEN 'active' THEN 'IN_PROGRESS'
           WHEN 'done' THEN 'COMPLETED'
           ELSE 'UNKNOWN'
       END,
       start_date
FROM player_quests;
```

### API Removals

| 0.x API | Status | 1.0 Replacement |
|---------|--------|-----------------|
| `QuestManager.createQuest()` | ❌ Removed | `QuestService.define()` |
| `Task` class | ❌ Removed | `Objective` class |
| `Reward` class | ❌ Removed | `RewardSet` + specific reward types |
| `questManager.assignQuest()` | ❌ Removed | `questService.startQuest()` |
| `task.markComplete()` | ❌ Removed | Event-driven progression |

### Migration Estimate

| Component | Complexity | Estimated Time |
|-----------|-----------|----------------|
| Package/Import Updates | Low | 1-2 hours |
| Configuration Migration | Low | 1 hour |
| Database Migration | Medium | 2-4 hours |
| Quest Definitions | High | 8-40 hours |
| Code Refactoring | Very High | 40-160 hours |

**Total**: 1-4 weeks for medium-sized project.

---

## Version 1.0 to 2.0

### Overview

Version 2.0 introduces architectural improvements, performance enhancements, and modern patterns. While significant, changes are more targeted than 0.x → 1.0.

### Timeline

- **1.0.0**: 2024-01-15
- **1.2.5 (Final 1.x)**: 2025-02-01
- **2.0.0**: 2025-03-01
- **Migration Support**: Until 2027-03-01

### Breaking Change Summary

| Category | Changes | Impact | Effort |
|----------|---------|--------|--------|
| Quest State Machine | Major refactor | High | High |
| Condition System | API redesign | High | Medium |
| NPC Framework | Structure change | Medium | Medium |
| Storage API | Type safety added | Medium | Low |
| Event System | Event sourcing | Medium | Low |
| Minimum Requirements | Java 25, newer DB | High | Low |

### 1. Quest State Machine Refactored

**Why**: More robust state management, better validation, audit trail.

**1.0 State Management**:
```java
// Simple state setter (1.0)
quest.setState(QuestState.COMPLETED);
quest.setState(QuestState.FAILED);

// No validation, no context, no history
```

**2.0 State Machine**:
```java
// Explicit state transitions with context (2.0)
TransitionResult result = quest.transitionTo(
    QuestState.COMPLETED,
    player,
    TransitionContext.builder()
        .reason("all_objectives_completed")
        .triggeredBy(TriggerSource.OBJECTIVE_COMPLETION)
        .metadata(Map.of(
            "completed_at", Instant.now(),
            "duration_seconds", 3600L
        ))
        .build()
);

// Validation, audit trail, rollback support
if (!result.isSuccessful()) {
    logger.warn("Transition failed: {}", result.getFailureReason());
}
```

**Migration Required**: 

```java
// Migration helper class
public class StateTransitionMigrator {
    
    // Pattern: Find all setState() calls
    @Deprecated
    public void oldSetState(Quest quest, QuestState state) {
        // Becomes:
        quest.transitionTo(state, 
            getCurrentPlayer(), 
            TransitionContext.auto());
    }
    
    // Automated migration via AST transformation
    public static void migrateFile(Path javaFile) {
        CompilationUnit cu = StaticJavaParser.parse(javaFile);
        
        cu.findAll(MethodCallExpr.class).forEach(call -> {
            if (call.getNameAsString().equals("setState")) {
                // Transform to transitionTo()
                call.setName("transitionTo");
                call.addArgument("player");
                call.addArgument("TransitionContext.auto()");
            }
        });
        
        Files.writeString(javaFile, cu.toString());
    }
}
```

**Impact**: All quest state changes must be updated.

**Estimated Effort**: 4-8 hours for 50 quests.

### 2. Condition Evaluation API Changed

**Why**: Performance, composability, better error handling.

**1.0 Conditions**:
```java
// Simple boolean evaluation (1.0)
public interface Condition {
    boolean evaluate(Player player);
}

// Usage
if (condition.evaluate(player)) {
    // Grant access
}

// Problems:
// - No context beyond player
// - No error information
// - Difficult to debug
// - Limited composability
```

**2.0 Conditions**:
```java
// Rich evaluation context (2.0)
public interface Condition {
    EvaluationResult evaluate(EvaluationContext context);
}

// EvaluationContext provides:
// - Player data
// - Quest context
// - Environment state
// - Temporal information
// - Custom variables

// Usage
EvaluationContext context = EvaluationContext.builder()
    .player(player)
    .quest(quest)
    .environment(gameEnvironment)
    .variable("time_of_day", timeOfDay)
    .build();

EvaluationResult result = condition.evaluate(context);

if (result.isSuccess()) {
    // Condition met
} else {
    // Rich failure information
    logger.debug("Condition failed: {} (reason: {})", 
        condition.getId(), 
        result.getFailureReason());
    
    // Can display to player
    player.sendMessage(result.getUserMessage());
}
```

**Migration Pattern**:

```java
// Before (1.0)
public class LevelCondition implements Condition {
    private final int requiredLevel;
    
    @Override
    public boolean evaluate(Player player) {
        return player.getLevel() >= requiredLevel;
    }
}

// After (2.0)
public class LevelCondition implements Condition {
    private final int requiredLevel;
    
    @Override
    public EvaluationResult evaluate(EvaluationContext context) {
        Player player = context.getPlayer();
        int currentLevel = player.getLevel();
        
        if (currentLevel >= requiredLevel) {
            return EvaluationResult.success();
        } else {
            return EvaluationResult.failure(
                String.format("Requires level %d (you are %d)", 
                    requiredLevel, currentLevel),
                Map.of("required", requiredLevel, "current", currentLevel)
            );
        }
    }
}
```

**Automated Migration**:

```bash
# Use AST-based transformer
./tools/migrate-conditions.sh \
  --source src/main/java/conditions/ \
  --add-context-parameter \
  --convert-return-type
```

**Impact**: All custom conditions must be updated.

**Estimated Effort**: 2-4 hours for 20 condition types.

### 3. NPC Framework Restructured

**Why**: Better dialogue management, state persistence, behavior trees.

**1.0 NPC Interaction**:
```java
// Direct dialogue display (1.0)
npc.showDialogue(player, "greeting_1");
npc.showDialogue(player, "quest_offer");

// Problems:
// - No session management
// - No state tracking
// - Limited branching
```

**2.0 NPC Interaction**:
```java
// Session-based dialogue (2.0)
DialogueSession session = npc.startDialogue(player);

session.show("greeting_1");

// Session maintains state
session.onChoice("accept_quest", () -> {
    questService.startQuest(player, "main_quest_1");
    session.show("quest_accepted");
});

session.onChoice("decline_quest", () -> {
    session.show("quest_declined");
});

// Session automatically persists
session.close();
```

**Migration Example**:

```java
// Migration wrapper for gradual transition
public class NPCInteractionAdapter {
    
    // Wrap 1.0 code to work with 2.0
    @Deprecated
    public void showDialogue_1_0_Compatible(NPC npc, Player player, String dialogueId) {
        // Create temporary session
        DialogueSession session = npc.startDialogue(player);
        session.show(dialogueId);
        
        // Auto-close after single message (1.0 behavior)
        session.setAutoClose(true);
    }
}
```

**Impact**: NPC interaction code needs updating.

**Estimated Effort**: 4-8 hours for 50 NPCs.

### 4. Storage API Type Safety

**Why**: Prevent ClassCastException, better IDE support, compile-time checking.

**1.0 Storage** (Unsafe):
```java
// Untyped storage (1.0)
storage.save("player_data", playerData);

// Unsafe cast required
PlayerData data = (PlayerData) storage.load("player_data");
// Risk: ClassCastException at runtime
```

**2.0 Storage** (Type-Safe):
```java
// Typed storage (2.0)
storage.save("player_data", playerData, PlayerData.class);

// Type-safe retrieval
PlayerData data = storage.load("player_data", PlayerData.class);
// Compile-time type checking

// Or use typed repository pattern
PlayerDataRepository repo = storage.getRepository(PlayerData.class);
repo.save("player_123", playerData);
PlayerData data = repo.load("player_123");
```

**Migration Tool**:

```java
// Automated migration via pattern matching
public class StorageMigrationTool {
    
    public void migrateStorageCalls(Path sourceFile) {
        String content = Files.readString(sourceFile);
        
        // Pattern: storage.load("key")
        // Replace with: storage.load("key", InferredType.class)
        
        Pattern loadPattern = Pattern.compile(
            "storage\\.load\\(\"([^\"]+)\"\\)"
        );
        
        Matcher matcher = loadPattern.matcher(content);
        StringBuffer sb = new StringBuffer();
        
        while (matcher.find()) {
            String key = matcher.group(1);
            String type = inferType(key); // Heuristic-based
            
            matcher.appendReplacement(sb, 
                String.format("storage.load(\"%s\", %s.class)", key, type));
        }
        matcher.appendTail(sb);
        
        Files.writeString(sourceFile, sb.toString());
    }
}
```

**Impact**: All storage operations need type parameters.

**Estimated Effort**: 2-3 hours for typical project.

### 5. Event Sourcing Introduction

**Why**: Complete audit trail, time-travel debugging, better analytics.

**New in 2.0** (Optional but recommended):

```java
// Enable event sourcing for quests
@EnableEventSourcing
public class TrackedQuest extends Quest {
    
    @Override
    public void completeObjective(UUID playerId, String objectiveId) {
        // Event is automatically recorded
        super.completeObjective(playerId, objectiveId);
        
        // Event store now contains:
        // {
        //   type: "OBJECTIVE_COMPLETED",
        //   questId: "quest_123",
        //   playerId: "uuid",
        //   objectiveId: "obj_1",
        //   timestamp: "2026-01-25T10:30:00Z",
        //   metadata: {...}
        // }
    }
}

// Query event history
List<QuestEvent> history = eventStore.getEvents(questId, playerId);

// Reconstruct state at any point in time
QuestState stateAt = eventStore.replayUntil(
    questId, 
    playerId, 
    Instant.parse("2026-01-20T00:00:00Z")
);
```

**Migration**: Optional, but recommended for new quests.

**Estimated Effort**: 4-8 hours to enable and test.

### 6. Minimum System Requirements Updated

**1.0 Requirements**:
- Java 17+
- MySQL 5.7+ or PostgreSQL 12+
- Minecraft 1.18+

**2.0 Requirements**:
- **Java 25+** (Required - uses newer language features and preview features)
- **MySQL 8.0+ or PostgreSQL 14+** (Required - uses newer SQL features)
- Minecraft 1.19+ (Recommended 1.20+)

**Migration Impact**:

```bash
# Update Java version
sudo apt update
sudo apt install openjdk-25-jdk

# Verify
java -version
# openjdk version "21.0.1"

# Update database (MySQL example)
sudo apt install mysql-server-8.0

# Migrate data
mysqldump --databases argonath > backup.sql
# Install MySQL 8.0
mysql < backup.sql
```

**Gradle Configuration**:

```gradle
// build.gradle
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

tasks.withType(JavaCompile) {
    options.release = 21
}
```

**Impact**: Infrastructure updates required.

**Estimated Effort**: 2-4 hours.

---

## API Renames and Removals

### Complete API Change Reference

#### Quest API

| 1.0 API | 2.0 API | Notes |
|---------|---------|-------|
| `quest.setState(state)` | `quest.transitionTo(state, player, context)` | Context required |
| `quest.getState()` | `quest.getCurrentState()` | Renamed for clarity |
| `quest.addObjective(obj)` | `quest.getObjectives().add(obj)` | Use builder pattern |
| `quest.setReward(reward)` | `quest.getRewards().add(reward)` | Multiple rewards supported |
| `quest.isActive()` | `quest.getCurrentState() == QuestState.IN_PROGRESS` | More explicit |
| `quest.complete()` | `quest.transitionTo(COMPLETED, ...)` | Use state machine |

#### Objective API

| 1.0 API | 2.0 API | Notes |
|---------|---------|-------|
| `objective.complete(player)` | `objective.attemptCompletion(player, context)` | Returns result |
| `objective.getProgress(player)` | `objective.getProgress(player).getValue()` | Wrapped in ProgressData |
| `objective.setProgress(player, value)` | `objective.updateProgress(player, value, source)` | Source tracking |

#### Condition API

| 1.0 API | 2.0 API | Notes |
|---------|---------|-------|
| `condition.evaluate(player)` | `condition.evaluate(context)` | Rich context |
| `condition.and(other)` | `condition.and(other)` | Unchanged |
| `condition.or(other)` | `condition.or(other)` | Unchanged |
| `condition.not()` | `condition.negate()` | Renamed |

#### NPC API

| 1.0 API | 2.0 API | Notes |
|---------|---------|-------|
| `npc.showDialogue(player, id)` | `npc.startDialogue(player).show(id)` | Session-based |
| `npc.setDialogue(id, text)` | `npc.getDialogueTree().setNode(id, node)` | Tree structure |
| `npc.interact(player)` | `npc.startInteraction(player)` | Renamed |

#### Storage API

| 1.0 API | 2.0 API | Notes |
|---------|---------|-------|
| `storage.save(key, value)` | `storage.save(key, value, type)` | Type parameter required |
| `storage.load(key)` | `storage.load(key, type)` | Type-safe |
| `storage.delete(key)` | `storage.delete(key)` | Unchanged |
| `storage.exists(key)` | `storage.contains(key)` | Renamed |

### Completely Removed APIs

These APIs were removed in 2.0 with **no direct replacement**:

| Removed API | Reason | Alternative |
|-------------|--------|-------------|
| `QuestManager.getAllQuests()` | Performance issues with large datasets | Use paginated `questService.findQuests(filter, page)` |
| `Quest.setPlayer(player)` | Quests are now player-agnostic | Store player data separately |
| `Objective.autoComplete()` | Unclear semantics | Use explicit completion with source |
| `Condition.evaluateAsync()` | Inconsistent async model | All evaluation is now async by default |
| `NPC.getLocation()` | NPCs can now be multi-location | Use `npc.getLocations()` |

---

## Configuration Format Changes

### 1.0 Configuration Format

```yaml
# config/argonath.yml (1.0)
argonath:
  quest:
    enabled: true
    max-active: 10
  
  storage:
    type: mysql
    host: localhost
    port: 3306
    database: argonath
    username: user
    password: pass
  
  performance:
    cache-size: 1000
```

### 2.0 Configuration Format

```yaml
# config/argonath.yml (2.0)
argonath:
  version: "2.0.0"  # NEW: Version tracking
  
  # CHANGED: Nested structure
  quest:
    enabled: true
    limits:  # NEW: Grouped settings
      max-active-per-player: 10
      max-total: 10000
    
    # NEW: State machine configuration
    state-machine:
      validate-transitions: true
      audit-enabled: true
  
  # CHANGED: Enhanced storage configuration
  storage:
    type: postgresql  # Note: type changed
    connection:
      host: localhost
      port: 5432
      database: argonath
      username: user
      password: ${ARGONATH_DB_PASSWORD}  # NEW: Environment variable support
    
    pool:  # NEW: Connection pool settings
      min-size: 5
      max-size: 20
      timeout-ms: 5000
  
  # NEW: Event sourcing
  event-sourcing:
    enabled: true
    store-type: database
    retention-days: 90
  
  # CHANGED: Enhanced performance settings
  performance:
    cache:
      quest-definitions: 1000
      player-data: 5000
      conditions: 2000
    
    async:
      core-pool-size: 4
      max-pool-size: 16
  
  # NEW: Plugin system
  plugins:
    enabled: true
    directory: "plugins/"
```

### Configuration Migration Tool

```bash
#!/bin/bash
# migrate-config.sh

./tools/config-migrator \
  --input config/argonath-1.0.yml \
  --output config/argonath-2.0.yml \
  --version 2.0 \
  --validate

# Validates:
# - Required fields present
# - Value types correct
# - No deprecated options
# - Environment variables set
```

### Breaking Configuration Changes

| 1.0 Setting | 2.0 Setting | Migration |
|-------------|-------------|-----------|
| `quest.max-active` | `quest.limits.max-active-per-player` | Rename + nest |
| `storage.host` | `storage.connection.host` | Nest under connection |
| `storage.password` | `storage.connection.password` | Move to env var |
| `performance.cache-size` | `performance.cache.quest-definitions` | Specific cache sizes |
| N/A | `event-sourcing.*` | New section, optional |
| N/A | `plugins.*` | New section, optional |

---

## Database Schema Migrations

### Schema Version Tracking

```sql
-- 2.0 introduces schema versioning
CREATE TABLE IF NOT EXISTS schema_migrations (
    id BIGSERIAL PRIMARY KEY,
    version VARCHAR(20) NOT NULL UNIQUE,
    description TEXT,
    applied_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    applied_by VARCHAR(100),
    checksum VARCHAR(64)
);

INSERT INTO schema_migrations (version, description)
VALUES ('2.0.0', 'Major upgrade from 1.x');
```

### 1.0 Schema

```sql
-- Quest tables (1.0)
CREATE TABLE quest_definitions (
    id VARCHAR(255) PRIMARY KEY,
    definition_data JSON NOT NULL,
    version INT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE quest_instances (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    state VARCHAR(50) NOT NULL,
    started_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    FOREIGN KEY (quest_id) REFERENCES quest_definitions(id)
);

CREATE TABLE quest_objectives (
    id BIGSERIAL PRIMARY KEY,
    instance_id BIGINT NOT NULL,
    objective_id VARCHAR(255) NOT NULL,
    progress INT DEFAULT 0,
    completed BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (instance_id) REFERENCES quest_instances(id)
);
```

### 2.0 Schema

```sql
-- Enhanced quest tables (2.0)

-- Quest definitions (enhanced)
CREATE TABLE quest_definitions (
    id VARCHAR(255) PRIMARY KEY,
    definition_data JSONB NOT NULL,  -- Changed JSON to JSONB for performance
    compiled_definition BYTEA,       -- NEW: Precompiled for faster loading
    version INT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,  -- NEW
    created_by VARCHAR(100),         -- NEW
    tags TEXT[]                      -- NEW: For categorization
);

-- Quest states with full history (NEW)
CREATE TABLE quest_states (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    previous_state VARCHAR(50),
    transition_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    triggered_by VARCHAR(255),       -- NEW: Audit trail
    transition_metadata JSONB,       -- NEW: Context data
    version INT NOT NULL DEFAULT 1,  -- NEW: Optimistic locking
    UNIQUE(quest_id, player_id)
);

-- State transition history (NEW)
CREATE TABLE quest_state_history (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    transitioned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    triggered_by VARCHAR(255),
    metadata JSONB,
    FOREIGN KEY (quest_id) REFERENCES quest_definitions(id)
);

-- Event sourcing (NEW)
CREATE TABLE quest_events (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    sequence_number BIGINT NOT NULL,
    aggregate_version INT NOT NULL,
    FOREIGN KEY (quest_id) REFERENCES quest_definitions(id)
);

CREATE INDEX idx_quest_events_quest_player 
ON quest_events(quest_id, player_id, sequence_number);

CREATE INDEX idx_quest_events_timestamp 
ON quest_events(timestamp);

-- Snapshots for performance (NEW)
CREATE TABLE quest_snapshots (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    state_data JSONB NOT NULL,
    snapshot_at_sequence BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(quest_id, player_id, snapshot_at_sequence)
);

-- NPC enhancements
ALTER TABLE npcs ADD COLUMN dialogue_state_machine JSONB;
ALTER TABLE npcs ADD COLUMN behavior_tree JSONB;
ALTER TABLE npcs ADD COLUMN metadata JSONB DEFAULT '{}';

-- Condition graph cache (NEW)
CREATE TABLE condition_graphs (
    id BIGSERIAL PRIMARY KEY,
    graph_id VARCHAR(255) NOT NULL UNIQUE,
    definition JSONB NOT NULL,
    compiled_form BYTEA,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Migration Script (1.0 → 2.0)

```sql
-- Full migration from 1.0 to 2.0
-- BACKUP YOUR DATABASE BEFORE RUNNING!

BEGIN;

-- 1. Add new columns to existing tables
ALTER TABLE quest_definitions 
    ADD COLUMN IF NOT EXISTS updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ADD COLUMN IF NOT EXISTS created_by VARCHAR(100),
    ADD COLUMN IF NOT EXISTS tags TEXT[],
    ADD COLUMN IF NOT EXISTS compiled_definition BYTEA;

-- 2. Convert JSON to JSONB for performance
ALTER TABLE quest_definitions 
    ALTER COLUMN definition_data TYPE JSONB USING definition_data::JSONB;

-- 3. Create new tables
CREATE TABLE IF NOT EXISTS quest_states (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    previous_state VARCHAR(50),
    transition_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    triggered_by VARCHAR(255),
    transition_metadata JSONB,
    version INT NOT NULL DEFAULT 1,
    UNIQUE(quest_id, player_id)
);

-- 4. Migrate existing state data
INSERT INTO quest_states (quest_id, player_id, current_state, transition_time)
SELECT quest_id, player_id, state, COALESCE(completed_at, started_at)
FROM quest_instances
ON CONFLICT (quest_id, player_id) DO NOTHING;

-- 5. Create history table
CREATE TABLE IF NOT EXISTS quest_state_history (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    transitioned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    triggered_by VARCHAR(255),
    metadata JSONB,
    FOREIGN KEY (quest_id) REFERENCES quest_definitions(id)
);

-- 6. Populate initial history from existing completions
INSERT INTO quest_state_history (quest_id, player_id, from_state, to_state, transitioned_at)
SELECT quest_id, player_id, 'NOT_STARTED', 'IN_PROGRESS', started_at
FROM quest_instances
WHERE state IN ('IN_PROGRESS', 'COMPLETED');

INSERT INTO quest_state_history (quest_id, player_id, from_state, to_state, transitioned_at)
SELECT quest_id, player_id, 'IN_PROGRESS', 'COMPLETED', completed_at
FROM quest_instances
WHERE state = 'COMPLETED' AND completed_at IS NOT NULL;

-- 7. Create event sourcing tables
CREATE TABLE IF NOT EXISTS quest_events (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    sequence_number BIGINT NOT NULL,
    aggregate_version INT NOT NULL,
    FOREIGN KEY (quest_id) REFERENCES quest_definitions(id)
);

CREATE INDEX IF NOT EXISTS idx_quest_events_quest_player 
ON quest_events(quest_id, player_id, sequence_number);

CREATE INDEX IF NOT EXISTS idx_quest_events_timestamp 
ON quest_events(timestamp);

CREATE TABLE IF NOT EXISTS quest_snapshots (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    state_data JSONB NOT NULL,
    snapshot_at_sequence BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(quest_id, player_id, snapshot_at_sequence)
);

-- 8. Update NPC tables
ALTER TABLE npcs 
    ADD COLUMN IF NOT EXISTS dialogue_state_machine JSONB,
    ADD COLUMN IF NOT EXISTS behavior_tree JSONB,
    ADD COLUMN IF NOT EXISTS metadata JSONB DEFAULT '{}';

-- 9. Create condition graph cache
CREATE TABLE IF NOT EXISTS condition_graphs (
    id BIGSERIAL PRIMARY KEY,
    graph_id VARCHAR(255) NOT NULL UNIQUE,
    definition JSONB NOT NULL,
    compiled_form BYTEA,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 10. Update schema version
INSERT INTO schema_migrations (version, description, applied_by)
VALUES ('2.0.0', 'Major upgrade: State machine, event sourcing, performance', 'migration-script')
ON CONFLICT (version) DO NOTHING;

COMMIT;

-- 11. Verify migration
DO $$
DECLARE
    v_quest_count_old INT;
    v_quest_count_new INT;
    v_player_count_old INT;
    v_player_count_new INT;
BEGIN
    SELECT COUNT(*) INTO v_quest_count_old FROM quest_instances;
    SELECT COUNT(*) INTO v_quest_count_new FROM quest_states;
    
    SELECT COUNT(DISTINCT player_id) INTO v_player_count_old FROM quest_instances;
    SELECT COUNT(DISTINCT player_id) INTO v_player_count_new FROM quest_states;
    
    RAISE NOTICE 'Old quest instances: %, New quest states: %', v_quest_count_old, v_quest_count_new;
    RAISE NOTICE 'Old players: %, New players: %', v_player_count_old, v_player_count_new;
    
    IF v_quest_count_old != v_quest_count_new THEN
        RAISE WARNING 'Quest count mismatch! Manual verification needed.';
    END IF;
    
    IF v_player_count_old != v_player_count_new THEN
        RAISE WARNING 'Player count mismatch! Manual verification needed.';
    END IF;
END $$;
```

---

## Migration Scripts and Examples

### Complete Migration Workflow

```bash
#!/bin/bash
# complete-migration-1.x-to-2.0.sh

set -e

echo "=== Argonath 1.x to 2.0 Migration ==="

# Configuration
BACKUP_DIR="backups/migration-$(date +%Y%m%d-%H%M%S)"
SRC_DIR="src/main/java"
CONFIG_DIR="config"

# Step 1: Pre-migration checks
echo "Step 1: Pre-migration checks..."
./tools/check-prerequisites.sh --target-version 2.0

# Step 2: Full backup
echo "Step 2: Creating backup..."
mkdir -p "$BACKUP_DIR"
./tools/backup-full.sh --output "$BACKUP_DIR"

# Step 3: Database migration
echo "Step 3: Migrating database..."
./tools/migrate-database.sh \
  --from 1.x \
  --to 2.0 \
  --backup "$BACKUP_DIR/db-backup.sql" \
  --verify

# Step 4: Configuration migration
echo "Step 4: Migrating configuration..."
./tools/migrate-config.sh \
  --input "$CONFIG_DIR/argonath-1.0.yml" \
  --output "$CONFIG_DIR/argonath-2.0.yml" \
  --backup

# Step 5: Code migration
echo "Step 5: Migrating code..."
./tools/migrate-code.sh \
  --source "$SRC_DIR" \
  --target-version 2.0 \
  --backup "$BACKUP_DIR/code-backup"

# Step 6: Dependency updates
echo "Step 6: Updating dependencies..."
./gradlew clean
sed -i 's/1\.[0-9]\.[0-9]/2.0.0/g' build.gradle
./gradlew dependencies --refresh-dependencies

# Step 7: Build and test
echo "Step 7: Building and testing..."
./gradlew build test integrationTest

# Step 8: Generate migration report
echo "Step 8: Generating report..."
./tools/generate-migration-report.sh \
  --backup-dir "$BACKUP_DIR" \
  --output migration-report.html

echo "=== Migration Complete ==="
echo "Backup location: $BACKUP_DIR"
echo "Report: migration-report.html"
echo ""
echo "Next steps:"
echo "1. Review migration report"
echo "2. Test in staging environment"
echo "3. Deploy to production when ready"
```

### Quest Definition Migration Example

```java
// Tool to migrate quest definitions
public class QuestDefinitionMigrator {
    
    public Quest20Definition migrate(Quest10Definition old) {
        return Quest20Definition.builder()
            .id(old.getId())
            .displayName(old.getDisplayName())
            .description(old.getDescription())
            
            // Migrate objectives
            .objectives(old.getObjectives().stream()
                .map(this::migrateObjective)
                .collect(Collectors.toList()))
            
            // Migrate rewards
            .rewards(migrateRewards(old.getRewards()))
            
            // NEW in 2.0: Initialize state machine
            .stateMachine(StateMachineDefinition.builder()
                .initialState(QuestState.NOT_STARTED)
                .allowedTransitions(getStandardTransitions())
                .build())
            
            // NEW in 2.0: Add metadata
            .metadata(Map.of(
                "migrated_from", "1.0",
                "migration_date", Instant.now().toString(),
                "original_id", old.getId()
            ))
            
            .build();
    }
    
    private Objective20 migrateObjective(Objective10 old) {
        return Objective20.builder()
            .id(old.getId())
            .type(old.getType())
            .description(old.getDescription())
            .targetValue(old.getTarget())
            
            // NEW in 2.0: Completion context
            .completionRequirements(CompletionRequirements.builder()
                .allowPartial(false)
                .validateSource(true)
                .build())
            
            .build();
    }
}
```

### Bulk Player Data Migration

```java
// Migrate all player quest data
public class PlayerDataMigrator {
    
    public void migrateAllPlayers(DataSource oldDb, DataSource newDb) {
        int batchSize = 1000;
        int offset = 0;
        
        while (true) {
            List<PlayerQuestData10> batch = fetchOldData(oldDb, offset, batchSize);
            
            if (batch.isEmpty()) {
                break;
            }
            
            List<PlayerQuestData20> migrated = batch.stream()
                .map(this::migratePlayerData)
                .collect(Collectors.toList());
            
            saveNewData(newDb, migrated);
            
            offset += batchSize;
            logger.info("Migrated {} players", offset);
        }
    }
    
    private PlayerQuestData20 migratePlayerData(PlayerQuestData10 old) {
        return PlayerQuestData20.builder()
            .playerId(old.getPlayerId())
            .questId(old.getQuestId())
            
            // Map old state to new state machine
            .currentState(mapState(old.getState()))
            
            // NEW in 2.0: Track when state was entered
            .stateEnteredAt(old.getUpdatedAt())
            
            // Migrate objective progress
            .objectiveProgress(old.getObjectiveProgress().entrySet().stream()
                .collect(Collectors.toMap(
                    Entry::getKey,
                    e -> ProgressData.of(e.getValue())
                )))
            
            // NEW in 2.0: Version for optimistic locking
            .version(1)
            
            .build();
    }
    
    private QuestState mapState(String oldState) {
        return switch (oldState) {
            case "active" -> QuestState.IN_PROGRESS;
            case "completed" -> QuestState.COMPLETED;
            case "failed" -> QuestState.FAILED;
            default -> QuestState.UNKNOWN;
        };
    }
}
```

---

## Compatibility Shims

### Purpose

Compatibility shims allow gradual migration by providing a compatibility layer between old and new APIs.

### Quest State Shim

```java
/**
 * Compatibility shim for Quest state management.
 * Allows 1.0 code to work with 2.0 quest system.
 * 
 * @deprecated Use Quest.transitionTo() directly in new code
 */
@Deprecated(since = "2.0", forRemoval = true)
public class QuestStateCompatibilityShim {
    
    private final Player defaultPlayer;
    
    /**
     * Emulates 1.0 setState() behavior
     */
    public void setState(Quest quest, QuestState state) {
        logger.warn("Using deprecated setState() - migrate to transitionTo()");
        
        try {
            quest.transitionTo(state, defaultPlayer, TransitionContext.auto());
        } catch (IllegalStateTransitionException e) {
            // 1.0 didn't validate transitions, so swallow exception
            logger.error("State transition validation failed (ignored in compat mode)", e);
        }
    }
    
    /**
     * Emulates 1.0 getState() behavior
     */
    public QuestState getState(Quest quest) {
        return quest.getCurrentState();
    }
}
```

### Condition Evaluation Shim

```java
/**
 * Allows 1.0 conditions to work in 2.0 system
 */
public class ConditionCompatibilityAdapter implements Condition {
    
    private final LegacyCondition10 legacyCondition;
    
    @Override
    public EvaluationResult evaluate(EvaluationContext context) {
        // Extract player from context
        Player player = context.getPlayer();
        
        // Call legacy method
        boolean result = legacyCondition.evaluate(player);
        
        // Convert to new result format
        if (result) {
            return EvaluationResult.success();
        } else {
            return EvaluationResult.failure(
                "Condition not met (legacy condition)",
                Collections.emptyMap()
            );
        }
    }
}
```

### NPC Dialogue Shim

```java
/**
 * Wraps 2.0 NPC dialogue system to provide 1.0-style API
 */
@Deprecated
public class NPCDialogueShim {
    
    private final Map<Player, DialogueSession> activeSessions = new HashMap<>();
    
    /**
     * Emulates 1.0 showDialogue() behavior
     */
    public void showDialogue(NPC npc, Player player, String dialogueId) {
        DialogueSession session = activeSessions.computeIfAbsent(
            player,
            p -> npc.startDialogue(p)
        );
        
        session.show(dialogueId);
        
        // Auto-close after showing (1.0 behavior)
        // In 2.0, sessions persist across multiple interactions
        session.setAutoClose(true);
    }
}
```

### Storage API Shim

```java
/**
 * Type-unsafe storage for backward compatibility
 */
@Deprecated
@SuppressWarnings("unchecked")
public class StorageCompatibilityShim {
    
    private final TypedStorage storage;
    
    /**
     * Emulates 1.0 save() without type parameter
     */
    public void save(String key, Object value) {
        // Try to infer type
        Class<?> type = value.getClass();
        
        try {
            storage.save(key, value, (Class) type);
        } catch (Exception e) {
            logger.error("Failed to save with inferred type", e);
            // Fallback: serialize to JSON
            storage.saveJson(key, value);
        }
    }
    
    /**
     * Emulates 1.0 load() without type parameter
     */
    public Object load(String key) {
        // Try common types
        for (Class<?> type : COMMON_TYPES) {
            try {
                return storage.load(key, type);
            } catch (ClassCastException | DeserializationException e) {
                // Wrong type, try next
            }
        }
        
        // Fallback: return raw JSON
        return storage.loadJson(key);
    }
    
    private static final List<Class<?>> COMMON_TYPES = List.of(
        String.class,
        Integer.class,
        Boolean.class,
        QuestProgress.class,
        PlayerData.class
    );
}
```

### Enabling Compatibility Mode

```yaml
# config/argonath.yml
argonath:
  version: "2.0.0"
  
  # Enable compatibility shims for gradual migration
  compatibility:
    enabled: true
    target-version: "1.2"  # Emulate 1.2 behavior
    
    features:
      quest-state-validation: false  # Disable validation (like 1.x)
      type-safe-storage: false       # Allow untyped storage
      dialogue-sessions: false       # Use legacy dialogue
      
    # Log compatibility usage for migration tracking
    logging:
      warn-on-shim-usage: true
      log-file: "logs/compatibility.log"
```

### Disabling Shims After Migration

```java
// Gradually disable shims as you migrate
@Configuration
public class MigrationConfig {
    
    @Bean
    @ConditionalOnProperty("argonath.compatibility.enabled")
    public QuestStateCompatibilityShim questStateShim() {
        logger.warn("Compatibility shim active - migrate code to 2.0 API");
        return new QuestStateCompatibilityShim();
    }
    
    // Once migration complete, remove @ConditionalOnProperty
    // and set argonath.compatibility.enabled=false
}
```

---

## Summary Tables

### Migration Effort by Component

| Component | 0.x → 1.0 | 1.0 → 2.0 | Automation Available |
|-----------|-----------|-----------|---------------------|
| Package Names | 1-2h | N/A | Yes (find/replace) |
| Configuration | 1h | 2-3h | Yes (config tool) |
| Database Schema | 2-4h | 3-6h | Yes (SQL scripts) |
| Quest Definitions | 8-40h | 4-8h | Partial (tool + manual) |
| Objective Logic | 4-16h | 2-4h | Partial |
| Condition Logic | 4-12h | 2-4h | Yes (code transformer) |
| NPC Interactions | 4-12h | 4-8h | Partial |
| Storage Calls | 2-4h | 2-3h | Yes (AST transformation) |
| Custom Code | Variable | Variable | No |

### Risk Assessment

| Migration Path | Risk Level | Recommended Approach |
|----------------|-----------|---------------------|
| 0.x → 1.0 | Very High | Rewrite, not upgrade |
| 1.0 → 1.1 | Low | Direct upgrade |
| 1.1 → 1.2 | Low | Direct upgrade |
| 1.0 → 2.0 | Medium-High | Staged migration |
| 1.2 → 2.0 | Medium | Staged migration |

---

## Conclusion

Breaking changes are inevitable in evolving software, but Argonath Systems provides:

✅ **Clear documentation** of all breaking changes  
✅ **Migration tools** to automate common updates  
✅ **Compatibility shims** for gradual migration  
✅ **Detailed examples** for manual migrations  
✅ **Database migration scripts** for schema updates  
✅ **Testing utilities** to validate migrations

**Best Practices**:
1. Always backup before migrating
2. Test in staging environment first
3. Use automated tools where available
4. Review breaking changes before starting
5. Plan for adequate time and resources
6. Monitor closely after production deployment

**Support Resources**:
- [Migration Overview](overview.md)
- [Upgrading Guide](upgrading.md)
- [Community Forum](https://community.argonath.systems)
- [GitHub Issues](https://github.com/argonath-systems/issues)
- [Discord](https://discord.gg/argonath)

---

*Last Updated: January 25, 2026*  
*Document Version: 1.0.0*  
*Covers: Argonath Systems 0.x through 2.1.x*
