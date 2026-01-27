# Dungeon & Instance System - Developer Guide

**Module:** `06-mod-dungeons-raids`  
**Package:** `com.argonathsystems.mod.dungeons`  
**Version:** 0.2.0  
**Dependencies:** `platform-core`, `framework-config`, `framework-condition`, `framework-npc`, `05-framework-quest`

## Overview

The Dungeon & Instance System provides procedurally generated instanced dungeons using **Wave Function Collapse (WFC)**, **Graph Grammar** progression systems, and Master Mask integration for theming. Features include boss encounters, multi-phase mechanics, loot distribution, and group management.

## Table of Contents

- [Architecture](#architecture)
- [Instance Management](#instance-management)
- [Wave Function Collapse](#wave-function-collapse)
- [Graph Grammar](#graph-grammar)
- [Boss Encounters](#boss-encounters)
- [Loot Distribution](#loot-distribution)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Integration Points](#integration-points)

---

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────────┐
│   Player/Group Interface                │
└─────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│   Instance Manager                      │
│  ┌─────────────────────────────────┐  │
│  │ • Instance Lifecycle            │  │
│  │ • Lockout Management            │  │
│  │ • Key Validation                │  │
│  │ • World Cloning                 │  │
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│   Generation Pipeline                   │
│  ┌─────────────────────────────────┐  │
│  │ 1. Graph Grammar                │  │ ← Logical Progression
│  │ 2. WFC Generator                │  │ ← Spatial Layout
│  │ 3. Prefab Placement             │  │ ← Visual Realization
│  │ 4. Entity Spawning              │  │ ← NPCs/Monsters
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────┐
│   Encounter System                      │
│  ┌─────────────────────────────────┐  │
│  │ • Boss Phase Handler            │  │
│  │ • Ability System                │  │
│  │ • Telegraph Rendering           │  │
│  │ • Loot Distribution             │  │
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Dependencies

**Required Frameworks:**
- `03-framework-config` - Dungeon/boss configuration (YAML)
- `04-framework-condition` - Boss ability conditions
- `02-framework-core` - Event system, logging

**Optional But Recommended:**
- `04-framework-npc` - NPC integration for friendly NPCs
- `05-framework-quest` - Quest objective integration
- `04-framework-currency` - Loot currency rewards
- `06-mod-loot-tables` - Loot table engine

---

## Instance Management

### Creating Instances

```java
import com.argonathsystems.mod.dungeons.instance.InstanceManager;
import com.argonathsystems.mod.dungeons.instance.DungeonInstance;
import com.argonathsystems.mod.dungeons.config.InstanceConfig;

public class DungeonExample {
    private final InstanceManager instanceManager;
    
    public void createDungeon(List<Player> party, String dungeonId) {
        // Load configuration
        InstanceConfig config = InstanceConfig.load(dungeonId);
        
        // Validate party
        if (party.size() < config.getMinPlayers()) {
            throw new IllegalArgumentException("Not enough players");
        }
        
        if (party.size() > config.getMaxPlayers()) {
            throw new IllegalArgumentException("Too many players");
        }
        
        // Check lockouts
        for (Player player : party) {
            if (instanceManager.isLockedOut(player, dungeonId)) {
                throw new LockoutException(
                    player.getName() + " is locked out until " + 
                    instanceManager.getLockoutExpiry(player, dungeonId)
                );
            }
        }
        
        // Create instance
        DungeonInstance instance = instanceManager.createInstance(
            config,
            party,
            System.currentTimeMillis() // seed
        );
        
        // Teleport party
        Location entrance = instance.getEntranceLocation();
        for (Player player : party) {
            player.teleport(entrance);
        }
    }
}
```

### Instance Lifecycle

```java
public class InstanceLifecycleExample {
    public void demonstrateLifecycle(DungeonInstance instance) {
        // 1. CREATED - Instance exists but not active
        assert instance.getState() == InstanceState.CREATED;
        
        // 2. ACTIVE - Players inside, encounters running
        instance.activate();
        assert instance.getState() == InstanceState.ACTIVE;
        
        // 3. Check if completed
        if (instance.allEncountersComplete()) {
            instance.complete();
            assert instance.getState() == InstanceState.COMPLETED;
            
            // Apply lockouts
            instanceManager.applyLockouts(instance);
        }
        
        // 4. EXPIRED - Timeout reached
        if (instance.getDurationSeconds() > instance.getMaxDuration()) {
            instance.expire();
            assert instance.getState() == InstanceState.EXPIRED;
        }
        
        // 5. DESTROYED - Cleanup and unload
        instanceManager.destroyInstance(instance.getId());
    }
}
```

### Lockout System

**Configuration:**
```yaml
# config/dungeons/crypt_tier1.yml
Lockout:
  Type: DAILY              # DAILY, WEEKLY, NONE
  ResetTime: "04:00:00 UTC"
  SharedWith: []           # Other dungeon IDs sharing lockout
```

**API Usage:**
```java
import com.argonathsystems.mod.dungeons.instance.LockoutManager;

public class LockoutExample {
    private final LockoutManager lockoutManager;
    
    public boolean canEnter(Player player, String dungeonId) {
        // Check if locked out
        if (lockoutManager.isLockedOut(player, dungeonId)) {
            Instant expiry = lockoutManager.getLockoutExpiry(player, dungeonId);
            player.sendMessage("Locked out until: " + expiry);
            return false;
        }
        
        return true;
    }
    
    public void applyLockout(DungeonInstance instance) {
        // Apply to all participants
        for (UUID playerId : instance.getParticipants()) {
            lockoutManager.applyLockout(
                playerId,
                instance.getConfig().getDungeonId(),
                instance.getConfig().getLockoutDuration()
            );
        }
    }
}
```

---

## Wave Function Collapse

### Concept

**Wave Function Collapse (WFC)** generates dungeons by:
1. Starting with all cells in "superposition" (can be any room type)
2. Collapsing cells one at a time based on **entropy** (possibilities)
3. Propagating **adjacency constraints** to neighbors
4. Repeating until all cells collapsed or contradiction detected

### Basic Usage

```java
import com.argonathsystems.mod.dungeons.generation.wfc.WFCDungeonGenerator;
import com.argonathsystems.mod.dungeons.generation.wfc.AdjacencyRules;

public class WFCExample {
    public DungeonLayout generateDungeon(int seed) {
        // Load adjacency rules
        AdjacencyRules rules = AdjacencyRules.load("crypt_rules.yml");
        
        // Create generator
        WFCDungeonGenerator generator = new WFCDungeonGenerator(rules);
        
        // Generate 10x3x10 grid (width, height, depth)
        DungeonLayout layout = generator.generate(seed, 10, 3, 10);
        
        // Layout contains room types at each coordinate
        RoomType entrance = layout.getRoomAt(5, 0, 0);
        assert entrance == RoomType.ROOM_SMALL;
        
        return layout;
    }
}
```

### Adjacency Rules

**File:** `config/wfc_rules/crypt_rules.yml`

```yaml
WFCAdjacencyRules:
  RuleSet: "crypt_undead"
  Version: 1.0.0
  
  # Room type definitions
  RoomTypes:
    WALL:
      Weight: 1.0
    
    CORRIDOR_NS:
      Weight: 3.0
      Description: "North-South corridor"
    
    CORRIDOR_EW:
      Weight: 3.0
      Description: "East-West corridor"
    
    ROOM_SMALL:
      Weight: 2.0
      Description: "Combat room"
    
    ROOM_LARGE:
      Weight: 0.5
      Description: "Boss/treasure room"
  
  # What can be next to what
  Rules:
    CORRIDOR_NS:
      North: [CORRIDOR_NS, ROOM_SMALL, ROOM_LARGE]
      South: [CORRIDOR_NS, ROOM_SMALL, ROOM_LARGE]
      East: [WALL]                    # Corridors have walls on sides
      West: [WALL]
      Up: [WALL]
      Down: [WALL]
    
    ROOM_SMALL:
      North: [CORRIDOR_NS, WALL]
      South: [CORRIDOR_NS, WALL]
      East: [CORRIDOR_EW, WALL]
      West: [CORRIDOR_EW, WALL]
      Up: [WALL]
      Down: [WALL]
```

### Advanced: Constrained WFC

Force specific cells to specific types (used with Graph Grammar):

```java
public class ConstrainedWFCExample {
    public DungeonLayout generateWithConstraints(GraphGrammar graph, int seed) {
        WFCDungeonGenerator generator = new WFCDungeonGenerator(rules);
        
        // Force entrance at (5, 0, 0)
        generator.forceCell(new GridPosition(5, 0, 0), RoomType.ROOM_SMALL);
        
        // Force boss arena at (5, 0, 9)
        generator.forceCell(new GridPosition(5, 0, 9), RoomType.ROOM_LARGE);
        
        // Generate with constraints
        DungeonLayout layout = generator.generate(seed, 10, 3, 10);
        
        // Entrance and boss guaranteed at specified locations
        return layout;
    }
}
```

### Error Handling

```java
try {
    DungeonLayout layout = generator.generate(seed, 10, 3, 10);
} catch (WFCContradictionException e) {
    // WFC failed - rules too restrictive
    // Retry with different seed or relax rules
    logger.warn("WFC contradiction: " + e.getMessage());
    
    // Fallback to simpler generation
    layout = fallbackGenerator.generate(seed);
}
```

---

## Graph Grammar

### Concept

**Graph Grammar** ensures dungeons have **solvable progression**:
- Entrance → Combat → Puzzle → Key → Locked Door → Treasure → Boss
- Prevents dead-ends, impossible puzzles, key-before-door errors

### Creating Progression Graphs

```java
import com.argonathsystems.mod.dungeons.generation.grammar.ProgressionGraphBuilder;
import com.argonathsystems.mod.dungeons.generation.grammar.ProgressionGraph;
import com.argonathsystems.mod.dungeons.generation.grammar.NodeType;

public class GraphGrammarExample {
    public ProgressionGraph createGraph() {
        return new ProgressionGraphBuilder()
            // Entrance (required)
            .addNode("entrance", NodeType.ENTRANCE)
                .setRoomType(RoomType.ROOM_SMALL)
                .setRequired(true)
            
            // First combat
            .addNode("combat1", NodeType.MONSTER_ROOM)
                .setRoomType(RoomType.ROOM_SMALL)
                .addSpawner("skeleton", 5)
            
            // Puzzle rewards key
            .addNode("puzzle", NodeType.PUZZLE_ROOM)
                .setRoomType(RoomType.ROOM_LARGE)
                .setPuzzleType("lever_sequence")
                .rewardsItem("silver_key")
            
            // Locked door consumes key
            .addNode("locked_door", NodeType.LOCKED_DOOR)
                .setRoomType(RoomType.CORRIDOR_NS)
                .requiresItem("silver_key")
                .consumesKey(true)
            
            // Boss arena (terminal)
            .addNode("boss", NodeType.BOSS_ARENA)
                .setRoomType(RoomType.ROOM_LARGE)
                .setBossId("undead_knight")
                .setTerminal(true)
            
            // Define progression flow
            .connect("entrance", "combat1")
            .connect("combat1", "puzzle")
            .connect("puzzle", "locked_door")
            .connect("locked_door", "boss")
            
            .build();
    }
}
```

### YAML Configuration

**File:** `config/dungeon_graphs/crypt_tier1.yml`

```yaml
ProgressionGraph:
  Id: "argonath:crypt_tier1"
  Name: "Abandoned Crypt - Tier 1"
  
  Nodes:
    entrance:
      Type: ENTRANCE
      RoomType: ROOM_SMALL
      Required: true
      PrefabPool: ["crypt_entrance_01", "crypt_entrance_02"]
    
    combat1:
      Type: MONSTER_ROOM
      RoomType: ROOM_SMALL
      Spawners:
        - EntityType: "skeleton"
          Count: 5
          Level: 10
    
    puzzle:
      Type: PUZZLE_ROOM
      RoomType: ROOM_LARGE
      PuzzleType: "lever_sequence"
      PuzzleConfig:
        LeverCount: 4
        CorrectSequence: [2, 4, 1, 3]
        TimeLimit: 120
      Reward:
        ItemId: "argonath:silver_key"
    
    locked_door:
      Type: LOCKED_DOOR
      RoomType: CORRIDOR_NS
      RequiredItem: "argonath:silver_key"
      ConsumeKey: true
    
    boss:
      Type: BOSS_ARENA
      RoomType: ROOM_LARGE
      BossId: "argonath:undead_knight"
      Terminal: true
  
  # Progression edges
  Edges:
    - From: entrance
      To: combat1
    
    - From: combat1
      To: puzzle
    
    - From: puzzle
      To: locked_door
      Condition: "has_item:argonath:silver_key"
    
    - From: locked_door
      To: boss
```

### Integration with WFC

```java
import com.argonathsystems.mod.dungeons.generation.grammar.GraphGrammarEngine;

public class IntegratedGenerationExample {
    public DungeonLayout generateDungeon(String graphId, int seed) {
        // Load graph
        ProgressionGraph graph = ProgressionGraph.load(graphId);
        
        // Generate with Graph Grammar + WFC
        GraphGrammarEngine engine = new GraphGrammarEngine();
        DungeonLayout layout = engine.generate(graph, seed);
        
        // Layout guarantees:
        // - All nodes placed
        // - Critical path exists (entrance → boss)
        // - Keys before locks
        // - Puzzles before rewards
        
        return layout;
    }
}
```

### Validation

```java
public class ValidationExample {
    public void validateDungeon(DungeonLayout layout, ProgressionGraph graph) {
        LayoutValidator validator = new LayoutValidator();
        
        ValidationResult result = validator.validate(layout, graph);
        
        if (!result.isValid()) {
            for (ValidationError error : result.getErrors()) {
                logger.error("Validation error: " + error.getMessage());
            }
            
            throw new DungeonGenerationException("Invalid dungeon layout");
        }
        
        // Check critical path
        assert result.hasCriticalPath();
        assert result.getEntranceNode() != null;
        assert result.getBossNode() != null;
        
        // Check all required nodes present
        for (String nodeId : graph.getRequiredNodes()) {
            assert layout.hasNode(nodeId);
        }
    }
}
```

---

## Boss Encounters

### Boss Definition

**File:** `config/bosses/undead_knight.yml`

```yaml
Boss:
  Id: "argonath:undead_knight"
  Name: "§4Sir Reginald the Fallen"
  Level: 15
  Health: 5000
  Armor: 50
  
  # Visual
  Model: "argonath:boss_undead_knight"
  Scale: 2.0
  
  # Multi-phase boss
  Phases:
    phase1:
      HealthRange: [100, 75]    # 100% to 75% HP
      
      Abilities:
        - Id: "cleave"
          Type: CONE_AOE
          Cooldown: 8.0
          Damage: 150
          Range: 5
          Angle: 90
          Telegraph:
            Type: "red_cone"
            Duration: 2.0       # 2s warning
        
        - Id: "summon_skeletons"
          Type: SUMMON
          Cooldown: 20.0
          Count: 3
          EntityType: "skeleton"
          Level: 10
    
    phase2:
      HealthRange: [75, 50]
      
      Transition:
        Animation: "roar"
        Duration: 3.0
        Invulnerable: true
        Message: "§cSir Reginald roars in fury!"
      
      Abilities:
        - Id: "cleave"
          Cooldown: 6.0         # Faster in phase 2
        
        - Id: "whirlwind"
          Type: CIRCLE_AOE
          Cooldown: 15.0
          Damage: 200
          Radius: 8
          Duration: 5.0
          Telegraph:
            Type: "red_circle_expanding"
            Duration: 3.0
    
    phase3:
      HealthRange: [50, 0]
      
      Transition:
        Animation: "dark_transformation"
        Duration: 5.0
        Invulnerable: true
        Message: "§5Darkness consumes Sir Reginald!"
        Effect: "shadow_aura"
      
      Abilities:
        - Id: "soul_drain"
          Type: RAID_WIDE
          Cooldown: 10.0
          Damage: 100           # Unavoidable
          HealsBoss: true
          HealAmount: 200
  
  # Loot
  LootTable: "argonath:boss_undead_knight"
  GuaranteedDrops:
    - ItemId: "argonath:fallen_knight_helm"
      Probability: 0.15
    
    - ItemId: "argonath:cursed_greatsword"
      Probability: 0.10
```

### Boss API

```java
import com.argonathsystems.mod.dungeons.encounter.BossEncounter;
import com.argonathsystems.mod.dungeons.encounter.BossPhase;

public class BossExample {
    private final BossEncounter encounter;
    
    public void startBoss(DungeonInstance instance) {
        // Load boss configuration
        BossConfig config = BossConfig.load("argonath:undead_knight");
        
        // Create encounter
        encounter = new BossEncounter(config, instance);
        
        // Register event listeners
        encounter.onPhaseChange(this::handlePhaseChange);
        encounter.onAbilityUse(this::handleAbilityUse);
        encounter.onDeath(this::handleBossDeath);
        
        // Start encounter
        encounter.start(instance.getPlayers());
    }
    
    private void handlePhaseChange(BossPhase oldPhase, BossPhase newPhase) {
        // Play transition animation
        newPhase.playTransitionAnimation();
        
        // Make boss invulnerable during transition
        if (newPhase.getTransitionConfig().isInvulnerable()) {
            encounter.setInvulnerable(true);
            
            // Re-enable after transition
            scheduler.runTaskLater(() -> {
                encounter.setInvulnerable(false);
            }, newPhase.getTransitionDuration());
        }
        
        // Broadcast message
        instance.broadcastMessage(newPhase.getTransitionMessage());
    }
    
    private void handleAbilityUse(BossAbility ability) {
        // Show telegraph
        if (ability.hasTelegraph()) {
            Telegraph telegraph = ability.getTelegraph();
            telegraph.show(instance.getPlayers());
            
            // Execute ability after telegraph
            scheduler.runTaskLater(() -> {
                ability.execute(encounter.getTargets());
            }, telegraph.getDuration());
        } else {
            // Instant ability
            ability.execute(encounter.getTargets());
        }
    }
    
    private void handleBossDeath(BossEncounter encounter) {
        // Mark encounter complete
        instance.completeEncounter(encounter.getId());
        
        // Distribute loot
        lootMaster.distributeLoot(
            instance.getPlayers(),
            encounter.getLootTable(),
            instance.getDifficulty()
        );
    }
}
```

### Telegraph System

Visual warnings for boss abilities:

```java
import com.argonathsystems.mod.dungeons.encounter.Telegraph;
import com.argonathsystems.mod.dungeons.encounter.TelegraphType;

public class TelegraphExample {
    public void showTelegraph(BossAbility ability, List<Player> players) {
        Telegraph telegraph = ability.getTelegraph();
        
        switch (telegraph.getType()) {
            case CONE_AOE:
                // Show red cone in boss facing direction
                showCone(
                    ability.getBoss().getLocation(),
                    ability.getBoss().getFacing(),
                    telegraph.getAngle(),
                    telegraph.getRange(),
                    Color.RED
                );
                break;
            
            case CIRCLE_AOE:
                // Show expanding red circle
                showExpandingCircle(
                    ability.getTargetLocation(),
                    telegraph.getRadius(),
                    telegraph.getDuration(),
                    Color.RED
                );
                break;
            
            case LINE_AOE:
                // Show beam line
                showLine(
                    ability.getBoss().getLocation(),
                    ability.getTargetLocation(),
                    telegraph.getWidth(),
                    Color.ORANGE
                );
                break;
        }
        
        // Play sound warning
        for (Player player : players) {
            player.playSound(
                player.getLocation(),
                Sound.BLOCK_BELL_USE,
                1.0f,
                0.8f
            );
        }
    }
}
```

---

## Loot Distribution

### Loot Systems

**Personal Loot:**
```java
import com.argonathsystems.mod.dungeons.loot.PersonalLoot;

public class PersonalLootExample {
    public void distributePersonalLoot(BossEncounter encounter) {
        PersonalLoot loot = new PersonalLoot(encounter.getLootTable());
        
        for (Player player : encounter.getPlayers()) {
            // Each player rolls independently
            List<ItemStack> items = loot.rollForPlayer(player);
            
            // Add to inventory
            for (ItemStack item : items) {
                player.getInventory().addItem(item);
                player.sendMessage("§aYou received: " + item.getDisplayName());
            }
        }
    }
}
```

**Group Loot (Need/Greed):**
```java
import com.argonathsystems.mod.dungeons.loot.LootMaster;
import com.argonathsystems.mod.dungeons.loot.LootRules;

public class GroupLootExample {
    private final LootMaster lootMaster;
    
    public void distributeGroupLoot(BossEncounter encounter) {
        List<ItemStack> drops = lootMaster.generateLoot(
            encounter.getLootTable(),
            encounter.getDifficulty()
        );
        
        for (ItemStack item : drops) {
            // Start loot roll
            LootRoll roll = lootMaster.startRoll(
                item,
                encounter.getPlayers(),
                LootRules.NEED_BEFORE_GREED
            );
            
            // Show UI to all players
            roll.showRollUI();
            
            // Wait for rolls (30s timeout)
            scheduler.runTaskLater(() -> {
                Player winner = roll.determineWinner();
                if (winner != null) {
                    winner.getInventory().addItem(item);
                    encounter.broadcastMessage(
                        winner.getName() + " won " + item.getDisplayName()
                    );
                }
            }, 30 * 20); // 30 seconds
        }
    }
}
```

### Configuration

```yaml
# config/loot_rules.yml
LootDistribution:
  Mode: PERSONAL          # PERSONAL, NEED_GREED, MASTER_LOOT
  
  PersonalLoot:
    RollPerPlayer: true
    MinimumGuaranteed: 1  # Each player gets at least 1 item
    
  NeedGreed:
    NeedRollBonus: 100    # Need rolls get +100 to roll
    RollTimeout: 30       # Seconds to roll
    PassByDefault: false  # Auto-pass if no response
  
  # Difficulty scaling
  DifficultyBonus:
    NORMAL: 1.0
    HEROIC: 1.3
    MYTHIC: 1.6
```

---

## Integration Points

### Framework Config Integration

```java
import com.argonathsystems.framework.config.ConfigFactory;

public class DungeonsPlugin {
    public void loadConfigurations() {
        ConfigFactory factory = new ConfigFactory(getDataFolder());
        
        // Load dungeon configs
        Map<String, DungeonConfig> dungeons = factory.loadAll(
            "dungeons/*.yml",
            DungeonConfig.class
        );
        
        // Load boss configs
        Map<String, BossConfig> bosses = factory.loadAll(
            "bosses/*.yml",
            BossConfig.class
        );
        
        // Load WFC rules
        Map<String, AdjacencyRules> wfcRules = factory.loadAll(
            "wfc_rules/*.yml",
            AdjacencyRules.class
        );
        
        // Register all
        for (var entry : dungeons.entrySet()) {
            dungeonRegistry.register(entry.getKey(), entry.getValue());
        }
    }
}
```

### Condition Framework Integration

**Boss Ability Conditions:**
```java
import com.argonathsystems.framework.condition.Condition;
import com.argonathsystems.framework.condition.ConditionContext;

public class BossAbilityCondition implements Condition {
    private final String abilityId;
    private final ConditionType type;
    
    @Override
    public boolean evaluate(ConditionContext context) {
        BossEncounter encounter = context.get("boss_encounter", BossEncounter.class);
        
        return switch (type) {
            case ON_COOLDOWN -> encounter.isAbilityOnCooldown(abilityId);
            case READY -> encounter.isAbilityReady(abilityId);
            case USED_THIS_PHASE -> encounter.wasAbilityUsedInPhase(abilityId);
        };
    }
}
```

**Usage in Boss Config:**
```yaml
Abilities:
  - Id: "enrage"
    Type: BUFF_SELF
    Conditions:
      - Type: "boss_health_percent"
        Operator: "<="
        Value: 25
      
      - Type: "boss_ability"
        Ability: "enrage"
        Condition: "not_used_this_phase"
```

### Quest Framework Integration

**Dungeon Quests:**
```yaml
Quest:
  Id: "clear_crypt_tier1"
  Name: "Clear the Abandoned Crypt"
  
  Objectives:
    - Type: "complete_dungeon"
      DungeonId: "argonath:crypt_tier1"
      Difficulty: "NORMAL"
  
  Conditions:
    - Type: "player_level"
      MinLevel: 10
  
  Rewards:
    Currency:
      - Type: "gold"
        Amount: 100
    
    Experience: 5000
```

**API:**
```java
import com.argonathsystems.framework.quest.QuestObjective;

public class DungeonQuestObjective implements QuestObjective {
    private final String dungeonId;
    
    @Override
    public boolean isComplete(Player player) {
        return instanceManager.hasCompleted(player, dungeonId);
    }
    
    @Override
    public void track(Player player) {
        // Listen for dungeon completion
        eventBus.on(DungeonCompleteEvent.class, event -> {
            if (event.getDungeonId().equals(dungeonId) &&
                event.getParticipants().contains(player.getUniqueId())) {
                this.setComplete(player);
            }
        });
    }
}
```

### NPC Framework Integration

**Dungeon NPCs:**
```java
import com.argonathsystems.framework.npc.NPC;
import com.argonathsystems.framework.npc.NPCManager;

public class DungeonNPCExample {
    public void spawnDungeonNPCs(DungeonInstance instance) {
        NPCManager npcManager = NPCManager.getInstance();
        
        // Spawn friendly NPC (questgiver)
        NPC questGiver = npcManager.createNPC("dungeon_questgiver")
            .setName("§aCaptured Villager")
            .setLocation(instance.getRoomLocation("entrance"))
            .setDialogTree(DialogTrees.DUNGEON_QUEST)
            .build();
        
        questGiver.spawn();
        instance.registerNPC(questGiver);
    }
}
```

---

## API Reference

### InstanceManager

```java
package com.argonathsystems.mod.dungeons.instance;

public interface InstanceManager {
    /**
     * Creates a new dungeon instance.
     * 
     * @param config Dungeon configuration
     * @param party Players in the party
     * @param seed Generation seed
     * @return Created instance
     * @throws LockoutException if any player is locked out
     * @throws IllegalArgumentException if party size invalid
     */
    DungeonInstance createInstance(
        InstanceConfig config,
        List<Player> party,
        long seed
    );
    
    /**
     * Destroys an instance and unloads world.
     * 
     * @param instanceId Instance UUID
     */
    void destroyInstance(UUID instanceId);
    
    /**
     * Checks if player is locked out.
     * 
     * @param player Player to check
     * @param dungeonId Dungeon ID
     * @return true if locked out
     */
    boolean isLockedOut(Player player, String dungeonId);
    
    /**
     * Gets lockout expiry time.
     * 
     * @param player Player to check
     * @param dungeonId Dungeon ID
     * @return Expiry instant, or null if not locked out
     */
    Instant getLockoutExpiry(Player player, String dungeonId);
}
```

### WFCDungeonGenerator

```java
package com.argonathsystems.mod.dungeons.generation.wfc;

public class WFCDungeonGenerator {
    /**
     * Generates dungeon layout.
     * 
     * @param seed Random seed
     * @param sizeX Grid width
     * @param sizeY Grid height
     * @param sizeZ Grid depth
     * @return Generated layout
     * @throws WFCContradictionException if unsolvable
     */
    public DungeonLayout generate(int seed, int sizeX, int sizeY, int sizeZ);
    
    /**
     * Forces a cell to a specific room type.
     * 
     * @param position Grid position
     * @param roomType Forced room type
     */
    public void forceCell(GridPosition position, RoomType roomType);
}
```

### BossEncounter

```java
package com.argonathsystems.mod.dungeons.encounter;

public interface BossEncounter {
    /**
     * Starts the encounter.
     * 
     * @param players Participating players
     */
    void start(List<Player> players);
    
    /**
     * Gets current phase.
     * 
     * @return Active phase
     */
    BossPhase getCurrentPhase();
    
    /**
     * Checks if ability is ready.
     * 
     * @param abilityId Ability identifier
     * @return true if off cooldown
     */
    boolean isAbilityReady(String abilityId);
    
    /**
     * Registers phase change callback.
     * 
     * @param callback Phase change handler
     */
    void onPhaseChange(BiConsumer<BossPhase, BossPhase> callback);
}
```

---

## See Also

- [World Generation System](world-generation.md)
- [Prefab Designer](../guides/prefab-designer.md)
- [Configuration Guide](../guides/configuration.md)
- [Boss Fight Tutorial](../guides/boss-fight-tutorial.md)
