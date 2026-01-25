# Quest Development Guide

A comprehensive guide to developing quests with the Argonath Framework Quest System.

## Table of Contents

1. [Introduction](#introduction)
2. [Quest Fundamentals](#quest-fundamentals)
3. [Quest Types](#quest-types)
4. [Quest Structure](#quest-structure)
5. [Creating Quests](#creating-quests)
6. [Quest Objectives](#quest-objectives)
7. [Quest Conditions](#quest-conditions)
8. [Quest Rewards](#quest-rewards)
9. [Quest Progression](#quest-progression)
10. [Advanced Patterns](#advanced-patterns)
11. [Testing and Debugging](#testing-and-debugging)
12. [Best Practices](#best-practices)
13. [Common Pitfalls](#common-pitfalls)
14. [Troubleshooting](#troubleshooting)

---

## Introduction

The Argonath Framework Quest System provides a powerful, flexible foundation for creating rich quest-driven gameplay experiences in Hytale. This guide covers everything from basic quest creation to advanced patterns and optimization techniques.

### Prerequisites

- Basic Java knowledge
- Familiarity with the Argonath Platform Core
- Understanding of [Core Concepts](../architecture/concepts.md)
- Completed [Quick Start Guide](../getting-started/quick-start.md)

### What You'll Learn

- Quest architecture and design patterns
- Creating different quest types
- Implementing objectives and conditions
- Managing quest progression and state
- Debugging and testing quests

---

## Quest Fundamentals

### What is a Quest?

A quest is a structured gameplay objective that guides player progression. In Argonath, quests are composed of:

- **Metadata**: ID, name, description, category
- **Objectives**: Tasks the player must complete
- **Conditions**: Requirements for acceptance and completion
- **Rewards**: Items, experience, or other benefits
- **Progression**: State tracking and event handling

### Quest Lifecycle

```
[Unavailable] → [Available] → [Active] → [Completed]
                      ↓
                 [Failed/Abandoned]
```

**States Explained:**
- **Unavailable**: Prerequisites not met
- **Available**: Can be accepted by player
- **Active**: Currently being pursued
- **Completed**: Successfully finished
- **Failed**: Conditions failed or abandoned

---

## Quest Types

### 1. Simple Quests

Single-objective quests with straightforward completion criteria.

```java
import com.argonath.framework.quest.Quest;
import com.argonath.framework.quest.QuestBuilder;
import com.argonath.framework.objective.ObjectiveType;
import com.argonath.framework.condition.ConditionType;

public class SimpleQuest {
    public static Quest createGatherQuest() {
        return new QuestBuilder("gather_wood")
            .withName("Gather Wood")
            .withDescription("Collect 10 oak logs for the carpenter")
            .withCategory("gathering")
            .addObjective(ObjectiveType.COLLECT)
                .item("oak_log")
                .amount(10)
                .build()
            .addReward()
                .experience(50)
                .item("copper_coin", 10)
                .build()
            .build();
    }
}
```

**Use Cases:**
- Tutorial quests
- Resource gathering
- Simple delivery tasks
- Introduction to game mechanics

---

### 2. Chain Quests

Multi-part quests where completion unlocks the next quest in the series.

```java
public class ChainQuest {
    public static Quest createChainPart1() {
        return new QuestBuilder("blacksmith_chain_1")
            .withName("The Blacksmith's Request - Part 1")
            .withDescription("Gather iron ore for the blacksmith")
            .withCategory("blacksmith_chain")
            .addObjective(ObjectiveType.COLLECT)
                .item("iron_ore")
                .amount(20)
                .build()
            .addReward()
                .experience(100)
                .build()
            .withNextQuest("blacksmith_chain_2")
            .build();
    }
    
    public static Quest createChainPart2() {
        return new QuestBuilder("blacksmith_chain_2")
            .withName("The Blacksmith's Request - Part 2")
            .withDescription("Deliver the ore to the blacksmith")
            .withCategory("blacksmith_chain")
            .addCondition(ConditionType.QUEST_COMPLETED)
                .questId("blacksmith_chain_1")
                .build()
            .addObjective(ObjectiveType.TALK_TO_NPC)
                .npcId("blacksmith_harold")
                .build()
            .addReward()
                .experience(150)
                .item("iron_sword")
                .build()
            .withNextQuest("blacksmith_chain_3")
            .build();
    }
    
    public static Quest createChainPart3() {
        return new QuestBuilder("blacksmith_chain_3")
            .withName("The Blacksmith's Request - Part 3")
            .withDescription("Test the new sword in combat")
            .withCategory("blacksmith_chain")
            .addCondition(ConditionType.QUEST_COMPLETED)
                .questId("blacksmith_chain_2")
                .build()
            .addObjective(ObjectiveType.KILL)
                .entityType("skeleton")
                .amount(5)
                .withRequiredItem("iron_sword")
                .build()
            .addReward()
                .experience(250)
                .item("gold_coin", 50)
                .reputation("blacksmith_guild", 100)
                .build()
            .build();
    }
}
```

**Best Practices for Chain Quests:**
- Keep individual parts focused and achievable
- Provide narrative continuity
- Scale rewards appropriately
- Allow abandonment of entire chain or just current part
- Consider level-gating later parts

---

### 3. Repeatable Quests

Quests that can be completed multiple times with cooldowns.

```java
public class RepeatableQuest {
    public static Quest createDailyBounty() {
        return new QuestBuilder("daily_bounty_wolves")
            .withName("Daily Bounty: Wolf Pack")
            .withDescription("Thin the wolf population in the northern woods")
            .withCategory("daily_bounty")
            .setRepeatable(true)
            .setCooldown(Duration.ofHours(24))
            .addObjective(ObjectiveType.KILL)
                .entityType("wolf")
                .amount(10)
                .inRegion("northern_woods")
                .build()
            .addReward()
                .experience(75)
                .item("wolf_pelt", 3)
                .currency(25)
                .build()
            .build();
    }
    
    public static Quest createWeeklyRaid() {
        return new QuestBuilder("weekly_raid_bandit_camp")
            .withName("Weekly Raid: Bandit Camp")
            .withDescription("Clear the bandit camp and defeat the leader")
            .withCategory("weekly_raid")
            .setRepeatable(true)
            .setCooldown(Duration.ofDays(7))
            .addCondition(ConditionType.MIN_LEVEL)
                .level(10)
                .build()
            .addObjective(ObjectiveType.KILL)
                .entityType("bandit")
                .amount(20)
                .build()
            .addObjective(ObjectiveType.KILL)
                .entityType("bandit_leader")
                .amount(1)
                .build()
            .addReward()
                .experience(500)
                .item("rare_weapon_token")
                .currency(100)
                .build()
            .withCompletionLimit(50) // Max 50 total completions
            .build();
    }
}
```

**Cooldown Strategies:**
- Daily: 24 hours (engagement)
- Weekly: 7 days (major content)
- Custom: Based on difficulty/rewards

---

### 4. Dynamic Quests

Procedurally generated or parameter-driven quests.

```java
public class DynamicQuest {
    private static final Random RANDOM = new Random();
    
    public static Quest createRandomBounty(String regionId) {
        String[] creatures = {"skeleton", "zombie", "spider", "wolf"};
        String creature = creatures[RANDOM.nextInt(creatures.length)];
        int amount = 5 + RANDOM.nextInt(16); // 5-20
        int reward = amount * 10;
        
        String questId = "bounty_" + creature + "_" + UUID.randomUUID();
        
        return new QuestBuilder(questId)
            .withName("Bounty: " + capitalize(creature) + " Threat")
            .withDescription("Eliminate " + amount + " " + creature + "s in the " + regionId)
            .withCategory("procedural_bounty")
            .setRepeatable(false)
            .setAutoGenerated(true)
            .addObjective(ObjectiveType.KILL)
                .entityType(creature)
                .amount(amount)
                .inRegion(regionId)
                .build()
            .addReward()
                .experience(reward)
                .currency(reward / 2)
                .build()
            .withExpirationTime(Duration.ofHours(2))
            .build();
    }
    
    public static Quest createTimedGatheringEvent(String itemType, int targetAmount) {
        return new QuestBuilder("timed_gathering_" + itemType)
            .withName("Timed Gathering: " + capitalize(itemType))
            .withDescription("Gather as much " + itemType + " as possible in 30 minutes")
            .withCategory("timed_event")
            .setRepeatable(false)
            .addObjective(ObjectiveType.COLLECT)
                .item(itemType)
                .amount(targetAmount)
                .build()
            .addTimeLimit(Duration.ofMinutes(30))
            .addReward()
                .experience(200)
                .scaledByCurrency(10) // Per item gathered
                .build()
            .withFailureOnTimeout(false) // Partial completion allowed
            .build();
    }
    
    private static String capitalize(String str) {
        return str.substring(0, 1).toUpperCase() + str.substring(1);
    }
}
```

---

### 5. Branching Quests

Quests with multiple paths based on player choices.

```java
public class BranchingQuest {
    public static Quest createMoralChoice() {
        return new QuestBuilder("moral_choice_bandits")
            .withName("The Bandit Problem")
            .withDescription("Decide how to deal with the bandits")
            .withCategory("story")
            .addObjective(ObjectiveType.CHOICE)
                .choice("spare_bandits", "Spare the bandits and redirect them")
                .choice("eliminate_bandits", "Eliminate the bandit threat")
                .choice("negotiate", "Negotiate a truce")
                .build()
            .onChoiceMade("spare_bandits", ctx -> {
                ctx.startQuest("spare_bandits_followup");
                ctx.modifyReputation("bandits", 50);
                ctx.modifyReputation("town_guard", -25);
            })
            .onChoiceMade("eliminate_bandits", ctx -> {
                ctx.startQuest("eliminate_bandits_followup");
                ctx.modifyReputation("town_guard", 50);
                ctx.modifyReputation("bandits", -100);
            })
            .onChoiceMade("negotiate", ctx -> {
                ctx.startQuest("negotiate_followup");
                ctx.modifyReputation("bandits", 25);
                ctx.modifyReputation("merchants", 25);
            })
            .addReward()
                .experience(300)
                .build()
            .build();
    }
}
```

---

## Quest Structure

### Quest Metadata

```java
public class QuestMetadata {
    private String id;              // Unique identifier
    private String name;            // Display name
    private String description;     // Quest description
    private String category;        // Categorization
    private QuestType type;         // Quest type enum
    private int recommendedLevel;   // Suggested level
    private String icon;            // Icon resource path
    private Set<String> tags;       // Searchable tags
    
    // Builder pattern
    public static class Builder {
        // Implementation
    }
}
```

### Quest Components

```java
public class QuestComponents {
    private List<QuestObjective> objectives;
    private List<QuestCondition> acceptConditions;
    private List<QuestCondition> failureConditions;
    private List<QuestReward> rewards;
    private QuestProgressTracker progressTracker;
    private QuestStateManager stateManager;
}
```

---

## Creating Quests

### Method 1: Builder Pattern (Recommended)

```java
public Quest createQuest() {
    return new QuestBuilder("my_quest_id")
        .withName("Quest Name")
        .withDescription("Quest description goes here")
        .withCategory("main_story")
        .setRecommendedLevel(5)
        .withIcon("textures/icons/quest_main.png")
        .addTag("story")
        .addTag("combat")
        
        // Accept conditions
        .addCondition(ConditionType.MIN_LEVEL)
            .level(5)
            .build()
        .addCondition(ConditionType.QUEST_COMPLETED)
            .questId("previous_quest")
            .build()
        
        // Objectives
        .addObjective(ObjectiveType.COLLECT)
            .item("rare_herb")
            .amount(5)
            .optional(false)
            .build()
        .addObjective(ObjectiveType.KILL)
            .entityType("forest_guardian")
            .amount(1)
            .build()
        
        // Rewards
        .addReward()
            .experience(250)
            .item("magic_staff")
            .currency(100)
            .reputation("mage_guild", 50)
            .build()
        
        .build();
}
```

### Method 2: Declarative JSON/YAML

```yaml
# quests/my_quest.yml
id: my_quest_id
name: "Quest Name"
description: "Quest description goes here"
category: main_story
recommended_level: 5
icon: "textures/icons/quest_main.png"
tags:
  - story
  - combat

conditions:
  - type: min_level
    level: 5
  - type: quest_completed
    quest_id: previous_quest

objectives:
  - type: collect
    item: rare_herb
    amount: 5
    optional: false
  - type: kill
    entity_type: forest_guardian
    amount: 1

rewards:
  experience: 250
  items:
    - id: magic_staff
      amount: 1
  currency: 100
  reputation:
    mage_guild: 50
```

```java
// Load from YAML
public class YamlQuestLoader {
    public Quest loadQuest(String yamlPath) {
        QuestConfig config = yamlParser.parse(yamlPath, QuestConfig.class);
        return questFactory.createFromConfig(config);
    }
}
```

### Method 3: Programmatic Creation

```java
public class ProgrammaticQuest extends Quest {
    public ProgrammaticQuest() {
        super("programmatic_quest");
        setName("Programmatic Quest");
        setDescription("Created through code");
        
        // Add components manually
        addObjective(new CollectObjective("iron_ore", 10));
        addReward(new ExperienceReward(100));
        
        // Custom logic
        onAccept(this::handleAccept);
        onComplete(this::handleComplete);
    }
    
    private void handleAccept(QuestContext ctx) {
        ctx.getPlayer().sendMessage("Quest accepted!");
        ctx.spawnGuide("quest_helper_npc");
    }
    
    private void handleComplete(QuestContext ctx) {
        ctx.getPlayer().sendMessage("Well done!");
        ctx.unlockRecipe("iron_sword");
    }
}
```

---

## Quest Objectives

### Objective Types

#### 1. Collection Objectives

```java
.addObjective(ObjectiveType.COLLECT)
    .item("diamond")
    .amount(5)
    .consumeOnComplete(true)  // Remove items when quest completes
    .allowCrafting(false)     // Must find, not craft
    .allowTrading(true)       // Can obtain through trading
    .build()
```

#### 2. Kill Objectives

```java
.addObjective(ObjectiveType.KILL)
    .entityType("dragon")
    .amount(1)
    .inRegion("dragon_peak")
    .withMinLevel(20)         // Entity must be at least level 20
    .requirePlayerKill(true)  // Must be killed by player
    .build()
```

#### 3. Talk/Interact Objectives

```java
.addObjective(ObjectiveType.TALK_TO_NPC)
    .npcId("village_elder")
    .requiredDialog("quest_dialog")
    .build()

.addObjective(ObjectiveType.INTERACT)
    .blockType("ancient_altar")
    .amount(1)
    .inRegion("temple_ruins")
    .build()
```

#### 4. Exploration Objectives

```java
.addObjective(ObjectiveType.DISCOVER)
    .location("hidden_cave")
    .radius(10)  // Within 10 blocks
    .build()

.addObjective(ObjectiveType.EXPLORE)
    .region("uncharted_lands")
    .percentage(75)  // Explore 75% of region
    .build()
```

#### 5. Crafting Objectives

```java
.addObjective(ObjectiveType.CRAFT)
    .item("iron_sword")
    .amount(3)
    .requireRecipe("iron_sword_recipe")
    .build()
```

#### 6. Custom Objectives

```java
public class CustomObjective extends QuestObjective {
    @Override
    public boolean isComplete(QuestContext ctx) {
        // Custom completion logic
        return ctx.getPlayer().getStat("custom_stat") >= 100;
    }
    
    @Override
    public float getProgress(QuestContext ctx) {
        return Math.min(1.0f, ctx.getPlayer().getStat("custom_stat") / 100.0f);
    }
    
    @Override
    public String getProgressText(QuestContext ctx) {
        int current = ctx.getPlayer().getStat("custom_stat");
        return current + " / 100";
    }
}
```

### Objective Progress Tracking

```java
public class ObjectiveProgressExample {
    public void trackProgress(Quest quest, Player player) {
        QuestProgress progress = quest.getProgress(player);
        
        for (QuestObjective objective : quest.getObjectives()) {
            float completion = objective.getProgress(player);
            String text = objective.getProgressText(player);
            
            System.out.println(objective.getName() + ": " + text + 
                             " (" + (completion * 100) + "%)");
        }
    }
}
```

---

## Quest Conditions

### Acceptance Conditions

Determine if a player can accept a quest.

```java
// Level requirements
.addCondition(ConditionType.MIN_LEVEL)
    .level(10)
    .build()

.addCondition(ConditionType.LEVEL_RANGE)
    .minLevel(10)
    .maxLevel(20)
    .build()

// Quest prerequisites
.addCondition(ConditionType.QUEST_COMPLETED)
    .questId("previous_quest")
    .build()

.addCondition(ConditionType.QUEST_NOT_COMPLETED)
    .questId("conflicting_quest")
    .build()

// Item requirements
.addCondition(ConditionType.HAS_ITEM)
    .item("special_key")
    .amount(1)
    .consumeOnAccept(true)
    .build()

// Reputation requirements
.addCondition(ConditionType.MIN_REPUTATION)
    .faction("mage_guild")
    .reputation(500)
    .build()

// Time-based conditions
.addCondition(ConditionType.TIME_OF_DAY)
    .timeRange(TimeRange.NIGHT)
    .build()

.addCondition(ConditionType.DAY_OF_WEEK)
    .daysAllowed(DayOfWeek.SATURDAY, DayOfWeek.SUNDAY)
    .build()

// Location conditions
.addCondition(ConditionType.IN_REGION)
    .regionId("starting_village")
    .build()

// Class/profession requirements
.addCondition(ConditionType.HAS_CLASS)
    .className("warrior")
    .build()
```

### Failure Conditions

Cause quest to fail if triggered.

```java
.addFailureCondition(ConditionType.NPC_DIES)
    .npcId("escort_target")
    .build()

.addFailureCondition(ConditionType.PLAYER_DIES)
    .build()

.addFailureCondition(ConditionType.TIME_LIMIT_EXCEEDED)
    .duration(Duration.ofMinutes(30))
    .build()

.addFailureCondition(ConditionType.ITEM_LOST)
    .item("quest_artifact")
    .build()

.addFailureCondition(ConditionType.REPUTATION_DROP)
    .faction("town_guard")
    .threshold(-100)
    .build()
```

### Custom Conditions

```java
public class CustomCondition implements QuestCondition {
    @Override
    public boolean isMet(QuestContext ctx) {
        Player player = ctx.getPlayer();
        
        // Custom logic
        return player.hasPermission("special.permission") &&
               player.getPlaytime() > Duration.ofHours(10).toMillis();
    }
    
    @Override
    public String getFailureMessage() {
        return "You must have played for at least 10 hours and have special permissions";
    }
}

// Usage
.addCondition(new CustomCondition())
    .build()
```

---

## Quest Rewards

### Basic Rewards

```java
.addReward()
    .experience(500)
    .item("diamond_sword", 1)
    .item("health_potion", 5)
    .currency(250)
    .build()
```

### Advanced Rewards

```java
.addReward()
    // Multiple items
    .items(Arrays.asList(
        new ItemStack("gold_coin", 50),
        new ItemStack("rare_gem", 3),
        new ItemStack("enchanted_armor")
    ))
    
    // Reputation changes
    .reputation("town_guard", 100)
    .reputation("thieves_guild", -50)
    
    // Title/achievement
    .title("Dragon Slayer")
    .achievement("first_dragon_kill")
    
    // Recipe unlocks
    .recipe("legendary_weapon_craft")
    
    // Skill points
    .skillPoints(5)
    
    // Custom rewards
    .custom("unlock_teleport", "town_square")
    
    .build()
```

### Choice Rewards

Allow players to choose one reward from options.

```java
.addRewardChoice()
    .maxChoices(1)
    .addOption()
        .item("warrior_sword")
        .name("Warrior's Path")
        .description("A powerful melee weapon")
        .build()
    .addOption()
        .item("mage_staff")
        .name("Mage's Path")
        .description("A magical staff")
        .build()
    .addOption()
        .item("ranger_bow")
        .name("Ranger's Path")
        .description("A precise ranged weapon")
        .build()
    .build()
```

### Scaled Rewards

Rewards that scale with level or performance.

```java
.addReward()
    .experience(player -> player.getLevel() * 50)
    .currency(player -> {
        int baseReward = 100;
        float performanceMultiplier = calculatePerformance(player);
        return (int)(baseReward * performanceMultiplier);
    })
    .build()
```

### Conditional Rewards

```java
.addReward()
    .experience(500)
    .build()
    
.addBonusReward()
    .condition(ctx -> ctx.getCompletionTime() < Duration.ofMinutes(10))
    .item("speed_completion_trophy")
    .title("Speed Runner")
    .build()
    
.addBonusReward()
    .condition(ctx -> !ctx.getPlayer().died())
    .item("flawless_completion_gem")
    .title("Flawless Victory")
    .build()
```

---

## Quest Progression

### State Management

```java
public class QuestStateExample {
    public void manageQuestState(Quest quest, Player player) {
        QuestState state = quest.getState(player);
        
        switch (state) {
            case UNAVAILABLE:
                // Check if conditions are now met
                if (quest.canAccept(player)) {
                    quest.setState(player, QuestState.AVAILABLE);
                }
                break;
                
            case AVAILABLE:
                // Waiting for player to accept
                break;
                
            case ACTIVE:
                // Update progress
                quest.updateProgress(player);
                
                // Check completion
                if (quest.isCompleted(player)) {
                    quest.complete(player);
                }
                
                // Check failure
                if (quest.hasFailed(player)) {
                    quest.fail(player);
                }
                break;
                
            case COMPLETED:
                // Quest finished
                if (quest.isRepeatable() && quest.canRepeat(player)) {
                    quest.setState(player, QuestState.AVAILABLE);
                }
                break;
                
            case FAILED:
                // Allow retry?
                if (quest.allowsRetry()) {
                    quest.reset(player);
                }
                break;
        }
    }
}
```

### Event Handling

```java
public class QuestEventHandlers {
    @EventHandler
    public void onQuestAccept(QuestAcceptEvent event) {
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        
        // Custom logic on accept
        player.sendMessage("Quest accepted: " + quest.getName());
        
        // Start timers, spawn NPCs, etc.
        if (quest.hasTimeLimit()) {
            startQuestTimer(quest, player);
        }
    }
    
    @EventHandler
    public void onQuestProgress(QuestProgressEvent event) {
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        QuestObjective objective = event.getObjective();
        
        // Update UI
        updateQuestTracker(player, quest);
        
        // Notify player
        if (objective.isComplete(player)) {
            player.sendMessage("Objective complete: " + objective.getName());
        }
    }
    
    @EventHandler
    public void onQuestComplete(QuestCompleteEvent event) {
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        
        // Award rewards
        quest.giveRewards(player);
        
        // Trigger next quest
        if (quest.hasNextQuest()) {
            Quest nextQuest = questManager.getQuest(quest.getNextQuestId());
            if (nextQuest.canAccept(player)) {
                nextQuest.setState(player, QuestState.AVAILABLE);
                player.sendMessage("New quest available: " + nextQuest.getName());
            }
        }
        
        // Unlock content
        unlockQuestRewards(player, quest);
    }
    
    @EventHandler
    public void onQuestFail(QuestFailEvent event) {
        Quest quest = event.getQuest();
        Player player = event.getPlayer();
        String reason = event.getFailureReason();
        
        player.sendMessage("Quest failed: " + reason);
        
        // Cleanup
        cleanupQuestState(player, quest);
    }
}
```

### Progress Tracking

```java
public class QuestProgressTracker {
    private final Map<UUID, Map<String, ObjectiveProgress>> playerProgress = new HashMap<>();
    
    public void updateObjective(Player player, Quest quest, String objectiveId, int amount) {
        ObjectiveProgress progress = getProgress(player, quest, objectiveId);
        progress.increment(amount);
        
        // Check if objective complete
        QuestObjective objective = quest.getObjective(objectiveId);
        if (progress.getCurrent() >= progress.getTarget()) {
            onObjectiveComplete(player, quest, objective);
        }
        
        // Check if all objectives complete
        if (quest.areAllObjectivesComplete(player)) {
            quest.complete(player);
        }
    }
    
    private ObjectiveProgress getProgress(Player player, Quest quest, String objectiveId) {
        return playerProgress
            .computeIfAbsent(player.getUUID(), k -> new HashMap<>())
            .computeIfAbsent(quest.getId() + ":" + objectiveId, 
                           k -> new ObjectiveProgress());
    }
}
```

---

## Advanced Patterns

### 1. Quest Phases

Break complex quests into distinct phases.

```java
public class PhasedQuest {
    public static Quest createPhasedQuest() {
        return new QuestBuilder("phased_investigation")
            .withName("The Investigation")
            .withDescription("Investigate the mysterious disappearances")
            
            // Phase 1: Gathering Information
            .addPhase("gathering_info")
                .withName("Gather Information")
                .addObjective(ObjectiveType.TALK_TO_NPC)
                    .npcId("witness_1")
                    .build()
                .addObjective(ObjectiveType.TALK_TO_NPC)
                    .npcId("witness_2")
                    .build()
                .addObjective(ObjectiveType.COLLECT)
                    .item("clue_fragment")
                    .amount(5)
                    .build()
                .build()
            
            // Phase 2: Investigation
            .addPhase("investigation")
                .withName("Investigate the Scene")
                .addCondition(ConditionType.PHASE_COMPLETED)
                    .phase("gathering_info")
                    .build()
                .addObjective(ObjectiveType.DISCOVER)
                    .location("crime_scene")
                    .build()
                .addObjective(ObjectiveType.INTERACT)
                    .blockType("evidence_marker")
                    .amount(3)
                    .build()
                .build()
            
            // Phase 3: Confrontation
            .addPhase("confrontation")
                .withName("Confront the Culprit")
                .addCondition(ConditionType.PHASE_COMPLETED)
                    .phase("investigation")
                    .build()
                .addObjective(ObjectiveType.KILL)
                    .entityType("culprit_boss")
                    .amount(1)
                    .build()
                .build()
            
            .addReward()
                .experience(1000)
                .item("detective_badge")
                .title("Master Detective")
                .build()
            .build();
    }
}
```

### 2. Shared Objectives

Multiple players contributing to the same objective.

```java
public class SharedObjectiveQuest {
    public static Quest createWorldBoss() {
        return new QuestBuilder("world_boss_dragon")
            .withName("Slay the Ancient Dragon")
            .withDescription("Work together to defeat the ancient dragon")
            .setSharedObjectives(true)
            .addObjective(ObjectiveType.KILL)
                .entityType("ancient_dragon")
                .amount(1)
                .sharedProgress(true)
                .contributionTracking(true)
                .build()
            .addReward()
                .scaledByContribution(true)
                .experience(1000)
                .item("dragon_scale", 1)
                .build()
            .build();
    }
}
```

### 3. Hidden Objectives

Objectives revealed during quest progression.

```java
public class HiddenObjectiveQuest {
    public static Quest createMysteryQuest() {
        return new QuestBuilder("mystery_quest")
            .withName("The Hidden Truth")
            .withDescription("Uncover the mystery")
            
            .addObjective(ObjectiveType.COLLECT)
                .item("ancient_scroll")
                .amount(1)
                .build()
            
            // Hidden objective - revealed when scroll is collected
            .addObjective(ObjectiveType.DECODE)
                .item("ancient_scroll")
                .hidden(true)
                .revealCondition(ctx -> 
                    ctx.hasCompletedObjective("collect_ancient_scroll"))
                .build()
            
            // Another hidden objective
            .addObjective(ObjectiveType.DISCOVER)
                .location("hidden_temple")
                .hidden(true)
                .revealCondition(ctx -> 
                    ctx.hasCompletedObjective("decode_ancient_scroll"))
                .build()
            
            .build();
    }
}
```

### 4. Escort Quests

```java
public class EscortQuest {
    public static Quest createEscort() {
        return new QuestBuilder("escort_merchant")
            .withName("Escort the Merchant")
            .withDescription("Safely escort the merchant to the next town")
            
            .onAccept(ctx -> {
                // Spawn NPC to escort
                NPC merchant = ctx.spawnNPC("merchant_gerald", ctx.getPlayer().getLocation());
                ctx.setQuestData("escort_npc_id", merchant.getId());
                merchant.followPlayer(ctx.getPlayer());
            })
            
            .addObjective(ObjectiveType.ESCORT)
                .npcId("merchant_gerald")
                .destination("next_town")
                .build()
            
            .addFailureCondition(ConditionType.NPC_DIES)
                .npcId("merchant_gerald")
                .build()
            
            .onComplete(ctx -> {
                // Despawn NPC
                String npcId = ctx.getQuestData("escort_npc_id");
                ctx.despawnNPC(npcId);
            })
            
            .onFail(ctx -> {
                // Cleanup
                String npcId = ctx.getQuestData("escort_npc_id");
                ctx.despawnNPC(npcId);
            })
            
            .build();
    }
}
```

---

## Testing and Debugging

### Quest Testing Framework

```java
public class QuestTester {
    private final QuestManager questManager;
    private final MockPlayer testPlayer;
    
    @Test
    public void testQuestAcceptance() {
        Quest quest = createTestQuest();
        
        // Test conditions not met
        assertFalse(quest.canAccept(testPlayer));
        
        // Meet conditions
        testPlayer.setLevel(10);
        testPlayer.completeQuest("prerequisite_quest");
        
        // Test can accept
        assertTrue(quest.canAccept(testPlayer));
        
        // Accept quest
        quest.accept(testPlayer);
        assertEquals(QuestState.ACTIVE, quest.getState(testPlayer));
    }
    
    @Test
    public void testObjectiveCompletion() {
        Quest quest = createTestQuest();
        quest.accept(testPlayer);
        
        // Simulate objective progress
        QuestObjective objective = quest.getObjectives().get(0);
        for (int i = 0; i < objective.getTargetAmount(); i++) {
            quest.updateObjective(testPlayer, objective.getId(), 1);
        }
        
        // Verify completion
        assertTrue(objective.isComplete(testPlayer));
    }
    
    @Test
    public void testQuestCompletion() {
        Quest quest = createTestQuest();
        quest.accept(testPlayer);
        
        // Complete all objectives
        for (QuestObjective objective : quest.getObjectives()) {
            completeObjective(testPlayer, objective);
        }
        
        // Verify quest completed
        assertTrue(quest.isCompleted(testPlayer));
        assertEquals(QuestState.COMPLETED, quest.getState(testPlayer));
        
        // Verify rewards given
        assertTrue(testPlayer.hasReceivedRewards(quest));
    }
}
```

### Debug Commands

```java
public class QuestDebugCommands {
    @Command("quest debug")
    @Permission("argonath.debug")
    public void debugQuest(Player player, String questId) {
        Quest quest = questManager.getQuest(questId);
        
        player.sendMessage("=== Quest Debug: " + questId + " ===");
        player.sendMessage("State: " + quest.getState(player));
        player.sendMessage("Can Accept: " + quest.canAccept(player));
        player.sendMessage("Is Completed: " + quest.isCompleted(player));
        
        player.sendMessage("\nObjectives:");
        for (QuestObjective obj : quest.getObjectives()) {
            float progress = obj.getProgress(player);
            String progressText = obj.getProgressText(player);
            player.sendMessage("  - " + obj.getName() + ": " + progressText + 
                             " (" + (progress * 100) + "%)");
        }
    }
    
    @Command("quest complete")
    @Permission("argonath.debug")
    public void forceComplete(Player player, String questId) {
        Quest quest = questManager.getQuest(questId);
        quest.forceComplete(player);
        player.sendMessage("Quest force completed: " + questId);
    }
    
    @Command("quest reset")
    @Permission("argonath.debug")
    public void resetQuest(Player player, String questId) {
        Quest quest = questManager.getQuest(questId);
        quest.reset(player);
        player.sendMessage("Quest reset: " + questId);
    }
}
```

### Logging

```java
public class QuestLogger {
    private static final Logger LOGGER = LoggerFactory.getLogger("QuestSystem");
    
    public void logQuestEvent(QuestEvent event) {
        String message = String.format(
            "[Quest:%s] [Player:%s] [Event:%s] %s",
            event.getQuest().getId(),
            event.getPlayer().getName(),
            event.getType(),
            event.getDetails()
        );
        
        LOGGER.info(message);
    }
    
    public void logQuestError(Quest quest, Player player, Exception e) {
        LOGGER.error("Quest error - Quest: {}, Player: {}", 
                    quest.getId(), player.getName(), e);
    }
}
```

---

## Best Practices

### 1. Quest Design

**DO:**
- ✅ Create clear, understandable objectives
- ✅ Provide appropriate rewards for effort
- ✅ Test quest flow thoroughly
- ✅ Include failure recovery options
- ✅ Make quest descriptions engaging
- ✅ Use consistent naming conventions
- ✅ Balance difficulty appropriately

**DON'T:**
- ❌ Create overly complex objective chains
- ❌ Make quests uncompletable
- ❌ Use confusing or ambiguous descriptions
- ❌ Forget to test edge cases
- ❌ Over-reward or under-reward
- ❌ Create impossible time limits

### 2. Performance

```java
// Good: Efficient progress tracking
public void updateProgress(Player player, String objectiveId, int amount) {
    QuestProgress progress = progressCache.get(player, objectiveId);
    if (progress != null) {
        progress.increment(amount);
        if (progress.isComplete()) {
            handleCompletion(player, objectiveId);
        }
    }
}

// Bad: Inefficient repeated queries
public void updateProgress(Player player, String objectiveId, int amount) {
    for (Quest quest : questManager.getAllQuests()) {
        if (quest.hasObjective(objectiveId)) {
            quest.updateObjective(player, objectiveId, amount);
            if (quest.getObjective(objectiveId).isComplete(player)) {
                // Check all objectives every time
                for (QuestObjective obj : quest.getObjectives()) {
                    if (!obj.isComplete(player)) {
                        return;
                    }
                }
                quest.complete(player);
            }
        }
    }
}
```

### 3. Modularity

```java
// Good: Reusable quest components
public class QuestComponents {
    public static QuestCondition levelRequirement(int level) {
        return new MinLevelCondition(level);
    }
    
    public static QuestObjective collectItems(String item, int amount) {
        return new CollectObjective(item, amount);
    }
    
    public static QuestReward standardReward(int experience, int currency) {
        return new RewardBuilder()
            .experience(experience)
            .currency(currency)
            .build();
    }
}

// Usage
Quest quest = new QuestBuilder("my_quest")
    .addCondition(QuestComponents.levelRequirement(10))
    .addObjective(QuestComponents.collectItems("iron", 20))
    .addReward(QuestComponents.standardReward(100, 50))
    .build();
```

### 4. Error Handling

```java
public class RobustQuestHandler {
    public void acceptQuest(Player player, Quest quest) {
        try {
            // Validate
            if (!quest.canAccept(player)) {
                String reason = quest.getAcceptanceFailureReason(player);
                player.sendMessage("Cannot accept quest: " + reason);
                return;
            }
            
            // Accept with rollback on failure
            QuestTransaction transaction = new QuestTransaction(player, quest);
            transaction.begin();
            
            try {
                quest.accept(player);
                transaction.commit();
            } catch (Exception e) {
                transaction.rollback();
                throw e;
            }
            
        } catch (Exception e) {
            LOGGER.error("Failed to accept quest", e);
            player.sendMessage("An error occurred. Please try again.");
        }
    }
}
```

---

## Common Pitfalls

### 1. Race Conditions

```java
// Problem: Multiple threads modifying quest state
public void updateQuest(Player player, Quest quest) {
    int progress = quest.getProgress(player);  // Thread 1 reads
    // Thread 2 updates here
    quest.setProgress(player, progress + 1);   // Thread 1 writes - data loss!
}

// Solution: Synchronization
public synchronized void updateQuest(Player player, Quest quest) {
    int progress = quest.getProgress(player);
    quest.setProgress(player, progress + 1);
}

// Better Solution: Atomic operations
public void updateQuest(Player player, Quest quest) {
    quest.incrementProgress(player);  // Atomic operation
}
```

### 2. Memory Leaks

```java
// Problem: Not cleaning up completed quests
public class QuestManager {
    private Map<UUID, List<Quest>> activeQuests = new HashMap<>();
    
    public void completeQuest(Player player, Quest quest) {
        quest.complete(player);
        // Forgot to remove from activeQuests!
    }
}

// Solution: Proper cleanup
public void completeQuest(Player player, Quest quest) {
    quest.complete(player);
    
    List<Quest> quests = activeQuests.get(player.getUUID());
    if (quests != null) {
        quests.remove(quest);
        if (quests.isEmpty()) {
            activeQuests.remove(player.getUUID());
        }
    }
}
```

### 3. Null Pointer Exceptions

```java
// Problem: Not checking for null
public void updateObjective(Player player, String questId, String objectiveId) {
    Quest quest = questManager.getQuest(questId);
    QuestObjective objective = quest.getObjective(objectiveId);  // NPE if quest is null!
    objective.updateProgress(player, 1);  // NPE if objective is null!
}

// Solution: Null checks
public void updateObjective(Player player, String questId, String objectiveId) {
    Quest quest = questManager.getQuest(questId);
    if (quest == null) {
        LOGGER.warn("Quest not found: {}", questId);
        return;
    }
    
    QuestObjective objective = quest.getObjective(objectiveId);
    if (objective == null) {
        LOGGER.warn("Objective not found: {} in quest {}", objectiveId, questId);
        return;
    }
    
    objective.updateProgress(player, 1);
}
```

---

## Troubleshooting

### Quest Won't Accept

**Check:**
1. All acceptance conditions are met
2. Player doesn't already have quest active
3. Quest isn't already completed (if not repeatable)
4. Cooldown has expired (if repeatable)

```java
public void diagnoseAcceptance(Player player, Quest quest) {
    if (!quest.canAccept(player)) {
        for (QuestCondition condition : quest.getAcceptConditions()) {
            if (!condition.isMet(player)) {
                player.sendMessage("Failed condition: " + condition.getFailureMessage());
            }
        }
    }
}
```

### Objectives Not Updating

**Check:**
1. Event listeners are registered
2. Objective types match the action
3. Progress tracking is enabled
4. No exceptions in objective update code

```java
@EventHandler
public void onItemPickup(ItemPickupEvent event) {
    Player player = event.getPlayer();
    ItemStack item = event.getItem();
    
    // Debug logging
    LOGGER.debug("Player {} picked up {}", player.getName(), item.getType());
    
    // Update all active quests
    for (Quest quest : questManager.getActiveQuests(player)) {
        quest.onItemPickup(player, item);
    }
}
```

### Quest Completion Issues

**Check:**
1. All objectives are complete
2. No failure conditions triggered
3. Completion handler is implemented
4. Rewards are properly configured

```java
public void debugCompletion(Player player, Quest quest) {
    player.sendMessage("Checking completion for: " + quest.getName());
    
    boolean allComplete = true;
    for (QuestObjective obj : quest.getObjectives()) {
        boolean complete = obj.isComplete(player);
        player.sendMessage("  " + obj.getName() + ": " + (complete ? "✓" : "✗"));
        if (!complete) allComplete = false;
    }
    
    player.sendMessage("All objectives complete: " + allComplete);
    player.sendMessage("Can complete: " + quest.canComplete(player));
}
```

### Performance Issues

**Symptoms:**
- Lag when updating quests
- Slow quest acceptance
- Memory usage grows over time

**Solutions:**

```java
// 1. Cache active quests per player
private final Map<UUID, Set<String>> playerActiveQuests = new ConcurrentHashMap<>();

public Set<Quest> getActiveQuests(Player player) {
    Set<String> questIds = playerActiveQuests.get(player.getUUID());
    if (questIds == null) return Collections.emptySet();
    
    return questIds.stream()
        .map(questManager::getQuest)
        .filter(Objects::nonNull)
        .collect(Collectors.toSet());
}

// 2. Batch updates
public void batchUpdateObjectives(Player player, Map<String, Integer> updates) {
    for (Quest quest : getActiveQuests(player)) {
        quest.batchUpdate(player, updates);
    }
}

// 3. Async quest checks
public CompletableFuture<Boolean> canAcceptAsync(Player player, Quest quest) {
    return CompletableFuture.supplyAsync(() -> quest.canAccept(player));
}
```

---

## Conclusion

This guide covered the fundamentals and advanced techniques of quest development in the Argonath Framework. For more information, see:

- [NPC Integration Guide](npc-integration.md) - Integrating quests with NPCs
- [UI Development Guide](ui-development.md) - Creating quest UIs
- [API Reference](../api/framework-quest.md) - Complete API documentation
- [Configuration Guide](configuration.md) - Quest configuration systems
- [Storage Guide](storage.md) - Persisting quest data

### Additional Resources

- [Examples Repository](https://github.com/argonath/quest-examples)
- [Community Discord](#) - Get help from other developers
- [Video Tutorials](#) - Step-by-step video guides

Happy quest building! 🎮
