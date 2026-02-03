# Version Upgrade Guide

## Introduction

This guide provides detailed procedures for upgrading between versions of Argonath Systems. Whether you're performing a minor version upgrade (e.g., 1.0.x to 1.1.x) or a major version upgrade (e.g., 1.x to 2.x), this document will help you navigate the process safely and efficiently.

## Table of Contents

1. [Version Compatibility](#version-compatibility)
2. [Upgrading from 1.0.x to 1.1.x](#upgrading-from-10x-to-11x)
3. [Upgrading from 1.x to 2.x](#upgrading-from-1x-to-2x)
4. [Breaking Changes by Version](#breaking-changes-by-version)
5. [Deprecated API Handling](#deprecated-api-handling)
6. [Automated Migration Tools](#automated-migration-tools)
7. [Step-by-Step Upgrade Procedures](#step-by-step-upgrade-procedures)
8. [Rollback Strategies](#rollback-strategies)

---

## Version Compatibility

### Version Numbering Scheme

Argonath Systems follows Semantic Versioning (SemVer):

```
MAJOR.MINOR.PATCH

Example: 2.1.3
├── 2 = Major version (breaking changes)
├── 1 = Minor version (new features, backward compatible)
└── 3 = Patch version (bug fixes, backward compatible)
```

### Compatibility Matrix

| From Version | To Version | Compatibility | Migration Complexity | Estimated Time |
|-------------|------------|---------------|---------------------|----------------|
| 1.0.x | 1.0.y | Full | None | Minutes |
| 1.0.x | 1.1.x | Backward Compatible | Low | 1-2 hours |
| 1.0.x | 1.2.x | Backward Compatible | Low | 2-4 hours |
| 1.x | 2.0.x | Breaking Changes | Medium-High | 1-3 days |
| 2.0.x | 2.1.x | Backward Compatible | Low | 1-2 hours |
| 1.x | 3.0.x | Major Breaking Changes | High | 1-2 weeks |

### Supported Upgrade Paths

**Direct Upgrades** (Supported):
- 1.0.x → 1.1.x → 1.2.x
- 2.0.x → 2.1.x → 2.2.x
- 1.2.x → 2.0.x
- 2.2.x → 3.0.x

**Incremental Upgrades** (Required):
- 1.0.x → 1.2.x → 2.0.x → 3.0.x
- Cannot skip major versions (e.g., 1.x → 3.x)

### Support Lifecycle

| Version | Release Date | End of Support | End of Life | Status |
|---------|-------------|----------------|-------------|--------|
| 1.0.x | 2024-01-15 | 2025-01-15 | 2026-01-15 | Maintenance |
| 1.1.x | 2024-06-01 | 2025-06-01 | 2026-06-01 | Maintenance |
| 1.2.x | 2024-12-01 | 2025-12-01 | 2026-12-01 | Maintenance |
| 2.0.x | 2025-03-01 | 2026-03-01 | 2027-03-01 | Active |
| 2.1.x | 2025-09-01 | 2026-09-01 | 2027-09-01 | Active |
| 3.0.x | 2026-04-01 | 2027-04-01 | 2028-04-01 | Development |

---

## Upgrading from 1.0.x to 1.1.x

### Overview

Version 1.1.x introduces new features while maintaining backward compatibility with 1.0.x. This is considered a **low-risk** upgrade.

### New Features in 1.1.x

- Enhanced condition system with new operators
- Improved NPC dialogue tree performance
- New text styling options (gradients, shadows)
- Quest template system
- Batch objective completion
- Advanced reward distribution

### Before You Begin

**Prerequisites**:
- [ ] Current version is 1.0.8 or higher (1.0.7 and below have known issues)
- [ ] Java 17 or higher
- [ ] Database backup completed
- [ ] Test environment available
- [ ] Read the [1.1.x changelog](../changelog/1.1.x.md)

**Estimated Downtime**: 15-30 minutes

### Step-by-Step Upgrade

#### Step 1: Prepare Your Environment

```bash
# 1. Create a backup
./backup-argonath.sh --version 1.0.x --output backups/pre-1.1-upgrade

# 2. Verify backup
./verify-backup.sh backups/pre-1.1-upgrade

# 3. Create rollback script
cat > rollback-1.1.sh << 'EOF'
#!/bin/bash
echo "Rolling back to 1.0.x..."
./stop-argonath.sh
./restore-backup.sh backups/pre-1.1-upgrade
./start-argonath.sh
echo "Rollback complete"
EOF
chmod +x rollback-1.1.sh
```

#### Step 2: Update Dependencies

```gradle
// build.gradle - Update version numbers
dependencies {
    // Platform Core
    implementation 'systems.argonath:platform-core:1.1.0'
    implementation 'systems.argonath:platform-sdk:1.1.0'
    
    // Frameworks
    implementation 'systems.argonath:framework-core:1.1.0'
    implementation 'systems.argonath:framework-accessor:1.1.0'
    implementation 'systems.argonath:framework-config:1.1.0'
    implementation 'systems.argonath:framework-storage:1.1.0'
    implementation 'systems.argonath:framework-text-styling:1.1.0'
    
    // Optional frameworks
    implementation 'systems.argonath:framework-condition:1.1.0'
    implementation 'systems.argonath:framework-npc:1.1.0'
    implementation 'systems.argonath:framework-objective:1.1.0'
    implementation 'systems.argonath:framework-quest:1.1.0'
    implementation 'systems.argonath:framework-ui:1.1.0'
}
```

```bash
# Update via command line
./gradlew clean
./gradlew dependencies --refresh-dependencies
./gradlew build
```

#### Step 3: Update Configuration Files

**1.1.x introduces optional new configuration options:**

```yaml
# config/argonath.yml - Add new optional settings

argonath:
  version: "1.1.0"
  
  # NEW in 1.1.x: Quest template system
  quest-templates:
    enabled: true
    cache-size: 100
    reload-on-change: true
  
  # NEW in 1.1.x: Batch processing
  batch-processing:
    enabled: true
    max-batch-size: 50
    timeout-ms: 5000
  
  # NEW in 1.1.x: Enhanced performance options
  performance:
    dialogue-tree-cache: true
    condition-evaluation-cache: true
    async-objective-completion: true
  
  # Existing settings remain unchanged
  storage:
    type: mysql
    # ... existing config
```

#### Step 4: Run Database Migrations

```bash
# 1.1.x includes automated schema updates
./argonath-migrate.sh --from 1.0.x --to 1.1.0

# Or manually:
mysql -u argonath -p argonath_db < migrations/1.0-to-1.1.sql
```

**Migration SQL** (automatically applied):

```sql
-- 1.0 to 1.1 Migration Script

-- Add quest template support
ALTER TABLE quests ADD COLUMN template_id VARCHAR(255) NULL AFTER quest_id;
ALTER TABLE quests ADD COLUMN template_version INT DEFAULT 1;

CREATE TABLE IF NOT EXISTS quest_templates (
    template_id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    definition JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    version INT DEFAULT 1
);

-- Add batch processing support
CREATE TABLE IF NOT EXISTS batch_operations (
    batch_id VARCHAR(36) PRIMARY KEY,
    operation_type VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP NULL,
    total_items INT DEFAULT 0,
    processed_items INT DEFAULT 0
);

-- Add indices for performance
CREATE INDEX idx_quests_template ON quests(template_id);
CREATE INDEX idx_batch_status ON batch_operations(status, created_at);

-- Update version tracking
INSERT INTO schema_version (version, applied_at) VALUES ('1.1.0', NOW());
```

#### Step 5: Update Your Code

**Deprecated API Usage** (still works, but update recommended):

```java
// Old way (1.0.x) - Still works in 1.1.x
quest.completeObjective(playerId, objectiveId);

// New way (1.1.x) - Preferred
quest.completeObjectives(playerId, List.of(objectiveId));
// OR for batch operations
quest.completeObjectivesBatch(playerId, objectiveIds);
```

**New Features** (optional to adopt):

```java
// 1. Use quest templates (NEW in 1.1.x)
QuestTemplate template = QuestTemplate.builder()
    .id("fetch_quest_template")
    .name("Generic Fetch Quest")
    .addObjective(ObjectiveTemplate.builder()
        .type(ObjectiveType.FETCH_ITEM)
        .parameter("item", "${quest.target_item}")
        .parameter("amount", "${quest.amount}")
        .build())
    .build();

Quest quest = Quest.fromTemplate(template)
    .withParameter("target_item", "golden_apple")
    .withParameter("amount", 5)
    .build();

// 2. Enhanced condition system (NEW in 1.1.x)
Condition condition = Condition.builder()
    .type("player_level")
    .operator(ComparisonOperator.GREATER_THAN_OR_EQUAL) // NEW operators
    .value(10)
    .and(Condition.builder()
        .type("has_permission")
        .value("vip.access")
        .build())
    .build();

// 3. Text styling enhancements (NEW in 1.1.x)
Component styledText = Component.text("Legendary Quest")
    .color(TextColor.gradient(0xFF0000, 0xFFFF00)) // Gradient support
    .decoration(TextDecoration.SHADOW, true) // Shadow support
    .build();
```

#### Step 6: Test in Development

```bash
# Start in development mode
./start-argonath.sh --mode development --version 1.1.0

# Run test suite
./gradlew test

# Run integration tests
./gradlew integrationTest

# Manual testing checklist:
# [ ] Quest creation and assignment
# [ ] Objective completion
# [ ] NPC interactions
# [ ] Reward distribution
# [ ] Condition evaluation
# [ ] UI rendering
# [ ] New template system (if using)
# [ ] Batch operations (if using)
```

#### Step 7: Deploy to Production

```bash
# 1. Put server in maintenance mode
./maintenance-mode.sh --enable --message "Upgrading to 1.1.0"

# 2. Stop Argonath
./stop-argonath.sh

# 3. Deploy new version
./deploy-argonath.sh --version 1.1.0

# 4. Run migrations (if not done in dev)
./argonath-migrate.sh --from 1.0.x --to 1.1.0

# 5. Start Argonath
./start-argonath.sh --version 1.1.0

# 6. Verify health
./health-check.sh

# 7. Disable maintenance mode
./maintenance-mode.sh --disable

# 8. Monitor logs
tail -f logs/argonath.log
```

### Post-Upgrade Validation

**Automated Checks**:

```bash
#!/bin/bash
# validate-1.1-upgrade.sh

echo "Validating 1.1.x upgrade..."

# Check version
VERSION=$(./argonath-cli.sh version)
if [[ "$VERSION" != "1.1."* ]]; then
    echo "ERROR: Version mismatch. Expected 1.1.x, got $VERSION"
    exit 1
fi

# Check database schema
SCHEMA_VERSION=$(mysql -u argonath -p -sN -e "SELECT version FROM schema_version ORDER BY applied_at DESC LIMIT 1")
if [[ "$SCHEMA_VERSION" != "1.1.0" ]]; then
    echo "ERROR: Schema version mismatch"
    exit 1
fi

# Test quest creation
./argonath-cli.sh quest create --test-mode

# Test NPC interaction
./argonath-cli.sh npc test --id test_npc

# Check performance
./performance-test.sh --duration 60

echo "Validation complete!"
```

**Manual Verification**:

1. [ ] Verify all quests are visible
2. [ ] Test quest assignment to player
3. [ ] Complete a quest objective
4. [ ] Complete an entire quest
5. [ ] Test NPC dialogue
6. [ ] Check reward distribution
7. [ ] Verify UI rendering
8. [ ] Test with real player accounts (if possible)

### Common Issues and Solutions

**Issue 1: Dependency Conflicts**

```
Error: Could not resolve systems.argonath:platform-core:1.1.0
```

**Solution**:
```bash
# Clear Gradle cache
./gradlew clean --refresh-dependencies

# Or manually clear cache
rm -rf ~/.gradle/caches/
./gradlew build
```

**Issue 2: Configuration Not Recognized**

```
Warning: Unknown configuration key 'quest-templates'
```

**Solution**: This is normal if you're using old configuration. The new keys are optional.

```yaml
# Update your config to acknowledge new version
argonath:
  version: "1.1.0"  # Add this to suppress warnings
```

**Issue 3: Database Migration Fails**

```
Error: Table 'quest_templates' already exists
```

**Solution**:
```bash
# Check if migration was partially applied
mysql -u argonath -p -e "SHOW TABLES LIKE 'quest_templates'"

# If exists, skip that part of migration
./argonath-migrate.sh --from 1.0.x --to 1.1.0 --skip-existing
```

---

## Upgrading from 1.x to 2.x

### Overview

Version 2.x is a **major upgrade** with significant architectural improvements and breaking changes. This requires careful planning and testing.

### Major Changes in 2.x

**Breaking Changes**:
- ⚠️ New quest state machine (incompatible with 1.x states)
- ⚠️ Refactored condition system API
- ⚠️ NPC framework restructured
- ⚠️ Storage format updated (requires migration)
- ⚠️ Deprecated APIs removed

**New Features**:
- 🎉 Advanced state machines for quests
- 🎉 Graph-based condition evaluation
- 🎉 Event sourcing for quest history
- 🎉 Plugin architecture for extensions
- 🎉 Real-time quest sharing/collaboration
- 🎉 Enhanced performance (2-3x faster)

### Before You Begin

**Prerequisites**:
- [ ] Currently on version 1.2.x or higher (must upgrade to 1.2.x first)
- [ ] Java 25 or higher (2.x requires Java 25+)
- [ ] PostgreSQL 14+ or MySQL 8+ (older versions not supported)
- [ ] Complete backup of all data
- [ ] Test environment that mirrors production
- [ ] Review all [breaking changes](breaking-changes.md#version-20x)
- [ ] Audit code for deprecated API usage
- [ ] Schedule maintenance window (2-4 hours recommended)

**Estimated Downtime**: 2-4 hours (depending on data volume)

### Pre-Upgrade Audit

**1. Check for Deprecated API Usage**:

```bash
# Run deprecation scanner
./argonath-scanner.sh --check-deprecated --version 2.0

# Output example:
# WARNING: Quest.setState() is deprecated
#   Location: src/main/java/com/example/CustomQuest.java:45
#   Replacement: Quest.transitionTo()
#
# WARNING: Condition.evaluate() signature changed
#   Location: src/main/java/com/example/CustomCondition.java:23
#   Replacement: Condition.evaluate(EvaluationContext)
```

**2. Analyze Data Migration Scope**:

```bash
# Analyze data to be migrated
./argonath-analyze.sh --target-version 2.0

# Output:
# Quests to migrate: 1,247
# NPC configurations: 456
# Player quest states: 15,623
# Estimated migration time: 45-60 minutes
# Required disk space: 2.3 GB
```

### Step-by-Step Upgrade

#### Phase 1: Preparation (Week Before)

```bash
# 1. Upgrade to latest 1.x version first
./gradlew dependencies --write-locks
./gradlew build

# 2. Run full backup
./backup-argonath.sh --full --output backups/pre-2.0-upgrade

# 3. Verify backup
./restore-backup.sh --dry-run backups/pre-2.0-upgrade

# 4. Clone production to staging
./clone-environment.sh --from production --to staging

# 5. Test upgrade on staging
./upgrade-to-2.0.sh --environment staging
```

#### Phase 2: Code Updates

**Update Dependencies**:

```gradle
// build.gradle
dependencies {
    // Update to 2.0
    implementation 'systems.argonath:platform-core:2.0.0'
    implementation 'systems.argonath:platform-sdk:2.0.0'
    
    // All frameworks updated to 2.0
    implementation 'systems.argonath:framework-core:2.0.0'
    implementation 'systems.argonath:framework-accessor:2.0.0'
    implementation 'systems.argonath:framework-config:2.0.0'
    implementation 'systems.argonath:framework-storage:2.0.0'
    implementation 'systems.argonath:framework-text-styling:2.0.0'
    implementation 'systems.argonath:framework-condition:2.0.0'
    implementation 'systems.argonath:framework-npc:2.0.0'
    implementation 'systems.argonath:framework-objective:2.0.0'
    implementation 'systems.argonath:framework-quest:2.0.0'
    implementation 'systems.argonath:framework-ui:2.0.0'
    
    // NEW in 2.0: Plugin framework
    implementation 'systems.argonath:framework-plugin:2.0.0'
    
    // NEW in 2.0: Event sourcing
    implementation 'systems.argonath:framework-event-sourcing:2.0.0'
}
```

**Update Java Version**:

```properties
# gradle.properties
org.gradle.java.home=/path/to/java21

# build.gradle
java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

**Migrate API Usage**:

```java
// ===== BREAKING CHANGE 1: Quest State Management =====

// OLD (1.x) - No longer works
quest.setState(QuestState.COMPLETED);

// NEW (2.0) - Required
quest.transitionTo(QuestState.COMPLETED, player, TransitionContext.builder()
    .triggeredBy("objective_completion")
    .metadata("completed_at", Instant.now())
    .build());


// ===== BREAKING CHANGE 2: Condition Evaluation =====

// OLD (1.x) - No longer works
boolean result = condition.evaluate(player);

// NEW (2.0) - Required
EvaluationContext context = EvaluationContext.builder()
    .player(player)
    .quest(quest)
    .environment(environment)
    .build();
boolean result = condition.evaluate(context);


// ===== BREAKING CHANGE 3: NPC Dialogue =====

// OLD (1.x) - No longer works
npc.showDialogue(player, "dialogue_id");

// NEW (2.0) - Required
DialogueSession session = npc.startDialogue(player);
session.show("dialogue_id");


// ===== BREAKING CHANGE 4: Objective Completion =====

// OLD (1.x) - No longer works
objective.complete(player);

// NEW (2.0) - Required
CompletionResult result = objective.attempt Completion(player, CompletionContext.builder()
    .source(CompletionSource.PLAYER_ACTION)
    .timestamp(Instant.now())
    .build());

if (result.isSuccess()) {
    // Handle success
}


// ===== BREAKING CHANGE 5: Storage API =====

// OLD (1.x) - No longer works
storage.save("key", data);
Object data = storage.load("key");

// NEW (2.0) - Required with type safety
storage.save("key", data, DataType.QUEST_PROGRESS);
QuestProgress data = storage.load("key", QuestProgress.class);


// ===== NEW FEATURE: Event Sourcing =====

// Track all quest events (optional but recommended)
questEventStore.append(QuestEvent.builder()
    .questId(quest.getId())
    .playerId(player.getId())
    .eventType(QuestEventType.OBJECTIVE_COMPLETED)
    .timestamp(Instant.now())
    .data(objectiveData)
    .build());

// Replay quest history
List<QuestEvent> history = questEventStore.getHistory(questId, playerId);
QuestState rebuiltState = QuestStateReconstructor.rebuild(history);
```

**Automated Migration Tool**:

```bash
# Run code migration tool
./argonath-migrate-code.sh --from 1.x --to 2.0 --source src/

# This will:
# 1. Scan for deprecated API usage
# 2. Attempt automatic refactoring
# 3. Generate a report of manual changes needed
# 4. Create a backup of original code
```

#### Phase 3: Configuration Migration

**Update Configuration Format**:

```yaml
# config/argonath.yml - 2.0 format

argonath:
  version: "2.0.0"
  
  # NEW: Plugin system
  plugins:
    enabled: true
    directory: "plugins/"
    auto-load: true
  
  # NEW: Event sourcing
  event-sourcing:
    enabled: true
    store-type: database  # or: file, redis
    retention-days: 90
    snapshot-interval: 100
  
  # UPDATED: Enhanced state machine
  quest-state-machine:
    enable-validation: true
    allow-state-reversion: false
    audit-transitions: true
  
  # UPDATED: Storage configuration
  storage:
    type: postgresql  # MySQL 8+ or PostgreSQL 14+
    connection:
      host: localhost
      port: 5432
      database: argonath
      username: argonath
      password: ${ARGONATH_DB_PASSWORD}
    
    # NEW: Connection pooling
    pool:
      min-size: 5
      max-size: 20
      timeout-ms: 5000
  
  # Performance optimizations (2.0 is 2-3x faster)
  performance:
    condition-graph-cache: true
    quest-state-cache-size: 1000
    async-event-processing: true
    batch-size: 100
```

#### Phase 4: Database Migration

**Major schema changes in 2.0**:

```bash
# Run migration tool
./argonath-db-migrate.sh --from 1.x --to 2.0.0

# Or manual migration:
psql -U argonath -d argonath < migrations/1.x-to-2.0.sql
```

**Migration Script** (excerpt):

```sql
-- 1.x to 2.0 Database Migration
-- WARNING: This is a DESTRUCTIVE migration. Backup required!

BEGIN TRANSACTION;

-- ===== Part 1: Schema Updates =====

-- Update quest_states table structure
ALTER TABLE quest_states RENAME TO quest_states_old;

CREATE TABLE quest_states (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    current_state VARCHAR(50) NOT NULL,
    previous_state VARCHAR(50),
    transition_time TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    triggered_by VARCHAR(255),
    metadata JSONB,
    version INT NOT NULL DEFAULT 1,
    UNIQUE(quest_id, player_id)
);

-- Migrate data from old format
INSERT INTO quest_states (quest_id, player_id, current_state, transition_time)
SELECT quest_id, player_id, state, updated_at
FROM quest_states_old;

-- ===== Part 2: Event Sourcing Tables =====

CREATE TABLE quest_events (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    sequence_number BIGINT NOT NULL,
    aggregate_version INT NOT NULL
);

CREATE INDEX idx_quest_events_quest_player ON quest_events(quest_id, player_id, sequence_number);
CREATE INDEX idx_quest_events_timestamp ON quest_events(timestamp);

-- Create snapshots table for performance
CREATE TABLE quest_snapshots (
    id BIGSERIAL PRIMARY KEY,
    quest_id VARCHAR(255) NOT NULL,
    player_id UUID NOT NULL,
    state_data JSONB NOT NULL,
    snapshot_at_sequence BIGINT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(quest_id, player_id, snapshot_at_sequence)
);

-- ===== Part 3: NPC Updates =====

ALTER TABLE npcs ADD COLUMN dialogue_state_machine JSONB;
ALTER TABLE npcs ADD COLUMN behavior_tree JSONB;
ALTER TABLE npcs ADD COLUMN metadata JSONB DEFAULT '{}';

-- ===== Part 4: Condition System =====

CREATE TABLE condition_graphs (
    id BIGSERIAL PRIMARY KEY,
    graph_id VARCHAR(255) NOT NULL UNIQUE,
    definition JSONB NOT NULL,
    compiled_form BYTEA,  -- Precompiled for performance
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ===== Part 5: Update Metadata =====

INSERT INTO schema_migrations (version, description, applied_at)
VALUES ('2.0.0', 'Major upgrade from 1.x', CURRENT_TIMESTAMP);

COMMIT;

-- ===== Post-Migration Validation =====

-- Verify record counts
SELECT 
    (SELECT COUNT(*) FROM quest_states_old) as old_count,
    (SELECT COUNT(*) FROM quest_states) as new_count;

-- Verify no data loss
SELECT 
    COUNT(DISTINCT quest_id, player_id) as unique_quests_old
FROM quest_states_old;

SELECT 
    COUNT(DISTINCT quest_id, player_id) as unique_quests_new
FROM quest_states;
```

#### Phase 5: Production Deployment

```bash
#!/bin/bash
# deploy-2.0.sh

set -e

echo "Starting Argonath 2.0 deployment..."

# 1. Final backup
echo "Creating final backup..."
./backup-argonath.sh --full --verify --output "backups/pre-2.0-$(date +%Y%m%d-%H%M%S)"

# 2. Enable maintenance mode
echo "Enabling maintenance mode..."
./maintenance-mode.sh --enable --message "Upgrading to Argonath 2.0 - ETA 2 hours"

# 3. Wait for active sessions to complete
echo "Waiting for active sessions..."
./wait-for-idle.sh --timeout 300

# 4. Stop services
echo "Stopping Argonath services..."
./stop-argonath.sh --graceful

# 5. Database migration
echo "Running database migration..."
./argonath-db-migrate.sh --from 1.x --to 2.0.0 --verbose

# 6. Verify migration
echo "Verifying migration..."
./verify-migration.sh --version 2.0.0

# 7. Deploy new code
echo "Deploying Argonath 2.0..."
./deploy.sh --version 2.0.0

# 8. Start services
echo "Starting Argonath 2.0..."
./start-argonath.sh --version 2.0.0

# 9. Health check
echo "Running health checks..."
./health-check.sh --wait --timeout 300

# 10. Smoke tests
echo "Running smoke tests..."
./smoke-tests.sh

# 11. Disable maintenance mode
echo "Disabling maintenance mode..."
./maintenance-mode.sh --disable

# 12. Monitor
echo "Monitoring for issues..."
./monitor.sh --duration 3600 --alert-on-error

echo "Deployment complete!"
```

### Post-Upgrade Validation

```bash
# Comprehensive validation suite
./validate-2.0-upgrade.sh

# Manual checks:
# [ ] Version is 2.0.x
# [ ] All quests visible
# [ ] Quest state transitions work
# [ ] NPC dialogues functional
# [ ] Conditions evaluate correctly
# [ ] Event sourcing recording events
# [ ] Performance improved (check metrics)
# [ ] No errors in logs
# [ ] Player progress preserved
```

### Performance Validation

```bash
# Compare performance metrics
./performance-compare.sh --baseline 1.x --current 2.0.x

# Expected improvements in 2.0:
# - Quest loading: 2-3x faster
# - Condition evaluation: 3-5x faster
# - Database queries: 40-60% reduction
# - Memory usage: 20-30% reduction
```

---

## Breaking Changes by Version

### Version 1.1.0

**No breaking changes** - Fully backward compatible with 1.0.x

**Deprecations** (still functional):
- `Quest.completeObjective()` → Use `Quest.completeObjectives()` (batch version)
- Single objective completion methods → Batch methods preferred

### Version 1.2.0

**Minor breaking changes**:
- Configuration file format updated (auto-migrated)
- Minimum Java version: 17 (was 11)

**Deprecations**:
- Old storage API → New type-safe storage API
- Legacy NPC interaction methods → New dialogue session API

### Version 2.0.0

**Major breaking changes**:
- Quest state management completely refactored
- Condition evaluation API changed
- NPC framework restructured
- Storage API requires type parameters
- Minimum Java version: 21
- Database schema incompatible with 1.x

See [Breaking Changes Reference](breaking-changes.md) for complete details.

---

## Deprecated API Handling

### Finding Deprecated APIs

```bash
# Scan your codebase
./argonath-scanner.sh --find-deprecated

# Output will show:
# - File and line number
# - Deprecated method/class
# - Recommended replacement
# - Version when deprecated
# - Version when removed
```

### Deprecation Timeline

| API | Deprecated In | Removed In | Replacement |
|-----|--------------|------------|-------------|
| `Quest.setState()` | 1.2.0 | 2.0.0 | `Quest.transitionTo()` |
| `Condition.evaluate(Player)` | 1.2.0 | 2.0.0 | `Condition.evaluate(EvaluationContext)` |
| `Storage.save(String, Object)` | 1.1.0 | 2.0.0 | `Storage.save(String, T, Class<T>)` |
| `NPC.showDialogue(Player, String)` | 1.2.0 | 2.0.0 | `NPC.startDialogue(Player).show(String)` |

### Migration Helper

```java
// Use compatibility shim for gradual migration (1.x only)
@EnableCompatibilityMode(version = "1.2")
public class MyQuestHandler {
    
    @Deprecated(since = "1.2", forRemoval = true)
    @ReplacedBy("transitionTo")
    public void setState(Quest quest, QuestState state) {
        // Old code still works with warning
        quest.setState(state);
    }
    
    // Gradual migration:new code
    public void transitionState(Quest quest, QuestState state, Player player) {
        quest.transitionTo(state, player, TransitionContext.auto());
    }
}
```

---

## Automated Migration Tools

### Quest Migrator

```bash
# Migrate quest definitions
./tools/migrate-quests.sh \
  --input quests/1.x/ \
  --output quests/2.0/ \
  --format yaml

# Options:
#   --dry-run: Show what would be changed
#   --backup: Create backup before migrating
#   --validate: Validate output
```

### Code Migrator

```bash
# Automated code refactoring
./tools/migrate-code.sh \
  --source src/main/java \
  --target-version 2.0 \
  --aggressive

# Options:
#   --conservative: Only safe changes
#   --aggressive: All possible changes
#   --interactive: Prompt for each change
```

### Data Migrator

```bash
# Migrate player data
./tools/migrate-data.sh \
  --from 1.2.x \
  --to 2.0.0 \
  --batch-size 1000 \
  --parallel 4

# Monitor progress:
./tools/migration-status.sh
```

---

## Step-by-Step Upgrade Procedures

### Procedure: Minor Version Upgrade (e.g., 1.1.0 → 1.1.5)

```bash
# 1. Backup (quick)
./backup-argonath.sh --incremental

# 2. Update dependencies
./gradlew clean build --refresh-dependencies

# 3. Deploy (hot swap possible for patches)
./deploy-argonath.sh --version 1.1.5 --hot-reload

# 4. Verify
./health-check.sh

# Total time: 5-10 minutes
```

### Procedure: Minor Feature Upgrade (e.g., 1.0.x → 1.1.x)

See [Upgrading from 1.0.x to 1.1.x](#upgrading-from-10x-to-11x) above.

### Procedure: Major Version Upgrade (e.g., 1.x → 2.x)

See [Upgrading from 1.x to 2.x](#upgrading-from-1x-to-2x) above.

---

## Rollback Strategies

### Automated Rollback

```bash
#!/bin/bash
# rollback.sh

VERSION_FROM=$1  # e.g., 2.0.0
VERSION_TO=$2    # e.g., 1.2.5

echo "Rolling back from $VERSION_FROM to $VERSION_TO..."

# 1. Stop current version
./stop-argonath.sh

# 2. Restore database
./restore-database.sh "backups/pre-${VERSION_FROM}"

# 3. Restore code
./restore-code.sh "$VERSION_TO"

# 4. Restore configuration
./restore-config.sh "backups/pre-${VERSION_FROM}"

# 5. Start previous version
./start-argonath.sh --version "$VERSION_TO"

# 6. Verify
./health-check.sh

echo "Rollback complete"
```

### Partial Rollback (Feature Flags)

```yaml
# config/features.yml
features:
  # Disable 2.0 features individually
  event-sourcing: false
  new-state-machine: false
  plugin-system: false
  
  # Use compatibility mode
  compatibility-mode:
    enabled: true
    target-version: "1.2"
```

### Blue-Green Deployment

```bash
# Keep both versions running
./deploy-blue-green.sh \
  --blue-version 1.2.x \
  --green-version 2.0.0 \
  --traffic-split 10  # 10% to green

# Gradually increase traffic
./adjust-traffic.sh --green-percent 50

# Full cutover
./cutover.sh --to green

# Rollback if needed
./cutover.sh --to blue
```

---

## Conclusion

Upgrading Argonath Systems requires careful planning but is straightforward with the right preparation:

- **Minor upgrades (1.0 → 1.1)**: Low risk, quick, minimal changes
- **Major upgrades (1.x → 2.x)**: Higher effort, significant benefits, requires testing
- **Always backup** before upgrading
- **Test thoroughly** in staging environment
- **Have a rollback plan** ready
- **Monitor closely** after upgrade

For assistance, consult:
- [Breaking Changes Reference](breaking-changes.md)
- [Migration Overview](overview.md)
- [Community Support](https://discord.gg/argonath)

---

*Last Updated: January 25, 2026*  
*Document Version: 1.0.0*  
*Covers Argonath Systems: 1.0.x - 2.1.x*
