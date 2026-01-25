# NPC Integration Guide

A comprehensive guide to creating, configuring, and integrating NPCs with the Argonath Framework.

## Table of Contents

1. [Introduction](#introduction)
2. [NPC Fundamentals](#npc-fundamentals)
3. [Creating NPCs](#creating-npcs)
4. [NPC Configuration](#npc-configuration)
5. [Dialog Systems](#dialog-systems)
6. [Quest Integration](#quest-integration)
7. [NPC Behaviors](#npc-behaviors)
8. [Pathfinding and Movement](#pathfinding-and-movement)
9. [NPC Interactions](#npc-interactions)
10. [Advanced Features](#advanced-features)
11. [Best Practices](#best-practices)
12. [Troubleshooting](#troubleshooting)

---

## Introduction

The Argonath Framework NPC system provides a flexible, extensible platform for creating interactive non-player characters. NPCs can serve as quest givers, merchants, companions, or environmental storytellers.

### Prerequisites

- Understanding of [Core Concepts](../architecture/concepts.md)
- Familiarity with [Quest Development](quest-development.md)
- Java programming knowledge

### What You'll Learn

- Creating and configuring NPCs
- Implementing dialog systems
- Integrating NPCs with quests
- NPC AI and behaviors
- Pathfinding and movement patterns

---

## NPC Fundamentals

### NPC Components

An NPC in Argonath consists of:

```java
public class NPC {
    private String id;                    // Unique identifier
    private String name;                  // Display name
    private NPCType type;                 // Quest giver, merchant, etc.
    private Location location;            // World position
    private NPCAppearance appearance;     // Visual representation
    private DialogTree dialogTree;        // Conversation structure
    private List<NPCBehavior> behaviors;  // AI behaviors
    private NPCState state;               // Current state
    private Map<String, Object> metadata; // Custom data
}
```

### NPC Types

```java
public enum NPCType {
    QUEST_GIVER,      // Provides quests
    MERCHANT,         // Trades items
    GUARD,            // Patrols and protects
    COMPANION,        // Follows player
    TRAINER,          // Teaches skills
    STORYTELLER,      // Provides lore
    AMBIENT,          // Background character
    CUSTOM            // Custom functionality
}
```

### NPC Lifecycle

```
[Create] → [Spawn] → [Active] → [Despawn] → [Destroyed]
                         ↓
                    [Respawn]
```

---

## Creating NPCs

### Method 1: Builder Pattern

```java
import com.argonath.framework.npc.NPC;
import com.argonath.framework.npc.NPCBuilder;
import com.argonath.framework.npc.NPCType;

public class NPCCreation {
    public static NPC createQuestGiver() {
        return new NPCBuilder("village_elder")
            .withName("Elder Theron")
            .withType(NPCType.QUEST_GIVER)
            .atLocation(new Location(world, 100, 64, 200))
            .withAppearance()
                .model("villager_elder")
                .skin("elder_theron")
                .scale(1.0f)
                .build()
            .withDialog("greeting", "Welcome, traveler!")
            .withQuests("elder_quest_1", "elder_quest_2")
            .withBehavior(NPCBehavior.IDLE)
            .withInteractionRange(5.0)
            .invulnerable(true)
            .build();
    }
    
    public static NPC createMerchant() {
        return new NPCBuilder("traveling_merchant")
            .withName("Marcus the Trader")
            .withType(NPCType.MERCHANT)
            .atLocation(new Location(world, 150, 64, 220))
            .withAppearance()
                .model("merchant")
                .skin("marcus")
                .equipment("backpack", "merchant_hat")
                .build()
            .withDialog("greeting", "Looking to buy or sell?")
            .withInventory()
                .addItem("health_potion", 10, 50)
                .addItem("mana_potion", 10, 50)
                .addItem("iron_sword", 1, 200)
                .currency(1000)
                .build()
            .withBehavior(NPCBehavior.WANDER)
                .wanderRadius(10)
                .build()
            .build();
    }
    
    public static NPC createGuard() {
        return new NPCBuilder("town_guard")
            .withName("Guard Captain Steel")
            .withType(NPCType.GUARD)
            .atLocation(new Location(world, 120, 64, 180))
            .withAppearance()
                .model("guard")
                .skin("captain")
                .equipment("iron_armor", "iron_sword", "shield")
                .build()
            .withDialog("greeting", "State your business!")
            .withBehavior(NPCBehavior.PATROL)
                .patrolPoints(
                    new Location(world, 120, 64, 180),
                    new Location(world, 140, 64, 180),
                    new Location(world, 140, 64, 200),
                    new Location(world, 120, 64, 200)
                )
                .build()
            .withCombatStats()
                .health(200)
                .damage(15)
                .defense(10)
                .build()
            .hostile(false)
            .build();
    }
}
```

### Method 2: JSON/YAML Configuration

```yaml
# npcs/village_elder.yml
id: village_elder
name: "Elder Theron"
type: QUEST_GIVER
location:
  world: overworld
  x: 100
  y: 64
  z: 200

appearance:
  model: villager_elder
  skin: elder_theron
  scale: 1.0

dialog:
  greeting: "Welcome, traveler!"
  quest_available: "I have a task that needs doing."
  quest_active: "How goes your quest?"
  quest_complete: "Excellent work! Here is your reward."
  farewell: "May your path be safe."

quests:
  - elder_quest_1
  - elder_quest_2

behaviors:
  - type: IDLE
    facing: SOUTH

interaction:
  range: 5.0
  
properties:
  invulnerable: true
  respawn: true
  respawn_time: 300
```

```java
// Load from YAML
public class NPCLoader {
    public NPC loadNPC(String yamlPath) {
        NPCConfig config = yamlParser.parse(yamlPath, NPCConfig.class);
        return npcFactory.createFromConfig(config);
    }
}
```

### Method 3: Programmatic Creation

```java
public class CustomNPC extends NPC {
    public CustomNPC() {
        super("custom_npc");
        setName("Custom NPC");
        setType(NPCType.CUSTOM);
        
        // Custom initialization
        registerInteractionHandler(this::handleInteraction);
        registerUpdateHandler(this::onUpdate);
    }
    
    private void handleInteraction(Player player) {
        // Custom interaction logic
        player.sendMessage("Custom interaction!");
    }
    
    private void onUpdate() {
        // Custom update logic
        if (shouldChangeState()) {
            setState(NPCState.ACTIVE);
        }
    }
}
```

---

## NPC Configuration

### Appearance Configuration

```java
public class NPCAppearanceConfig {
    public static NPCAppearance createWarriorAppearance() {
        return new NPCAppearanceBuilder()
            .model("warrior")
            .skin("warrior_veteran")
            .scale(1.1f)
            .equipment()
                .helmet("iron_helmet")
                .chestplate("iron_chestplate")
                .leggings("iron_leggings")
                .boots("iron_boots")
                .mainHand("iron_sword")
                .offHand("shield")
                .build()
            .particleEffect("strength_aura")
            .glowing(false)
            .nametagVisible(true)
            .build();
    }
}
```

### Interaction Configuration

```java
public class NPCInteractionConfig {
    public static void configureInteraction(NPC npc) {
        npc.setInteractionRange(5.0);
        npc.setInteractionCooldown(Duration.ofSeconds(1));
        
        // Right-click interaction
        npc.onRightClick(player -> {
            if (npc.hasQuestsAvailable(player)) {
                npc.showQuestDialog(player);
            } else {
                npc.showDefaultDialog(player);
            }
        });
        
        // Left-click interaction (if enabled)
        npc.onLeftClick(player -> {
            if (npc.isMerchant()) {
                npc.openTradeWindow(player);
            }
        });
        
        // Proximity interaction
        npc.onPlayerNearby(player -> {
            if (npc.isFirstEncounter(player)) {
                npc.triggerGreeting(player);
            }
        });
    }
}
```

### Combat Configuration

```java
public class NPCCombatConfig {
    public static void configureCombat(NPC npc) {
        npc.setCombatStats(new CombatStats()
            .health(150)
            .maxHealth(150)
            .damage(12)
            .defense(8)
            .attackSpeed(1.0)
            .movementSpeed(0.25)
        );
        
        npc.setCombatBehavior(new CombatBehavior()
            .attackRange(3.0)
            .detectionRange(15.0)
            .chaseRange(20.0)
            .retreatHealthPercent(0.3)
            .callForHelp(true)
            .helpRadius(10.0)
        );
        
        npc.setHostile(false);
        npc.setRetaliates(true);
    }
}
```

---

## Dialog Systems

### Simple Dialogs

```java
public class SimpleDialog {
    public static void createSimpleDialog(NPC npc) {
        DialogTree dialog = new DialogTreeBuilder()
            .addNode("greeting")
                .text("Hello, traveler!")
                .addResponse("Hello")
                    .goToNode("main_menu")
                    .build()
                .build()
            
            .addNode("main_menu")
                .text("How can I help you?")
                .addResponse("Tell me about this place")
                    .goToNode("about_place")
                    .build()
                .addResponse("Goodbye")
                    .endDialog()
                    .build()
                .build()
            
            .addNode("about_place")
                .text("This is a peaceful village...")
                .addResponse("I see")
                    .goToNode("main_menu")
                    .build()
                .build()
            
            .build();
        
        npc.setDialogTree(dialog);
    }
}
```

### Conditional Dialogs

```java
public class ConditionalDialog {
    public static DialogTree createConditionalDialog() {
        return new DialogTreeBuilder()
            .addNode("greeting")
                .text(ctx -> {
                    Player player = ctx.getPlayer();
                    int rep = player.getReputation("village");
                    
                    if (rep >= 100) {
                        return "Welcome, honored friend!";
                    } else if (rep >= 0) {
                        return "Hello, traveler.";
                    } else {
                        return "What do you want?";
                    }
                })
                .addResponse("Talk")
                    .condition(ctx -> ctx.getPlayer().getReputation("village") >= 0)
                    .goToNode("friendly_talk")
                    .build()
                .addResponse("Leave")
                    .endDialog()
                    .build()
                .build()
            
            .addNode("friendly_talk")
                .text("I'm glad to see you!")
                .addResponse("About quests...")
                    .condition(ctx -> ctx.getNPC().hasQuestsAvailable(ctx.getPlayer()))
                    .goToNode("quest_menu")
                    .build()
                .addResponse("Goodbye")
                    .endDialog()
                    .build()
                .build()
            
            .build();
    }
}
```

### Quest Dialogs

```java
public class QuestDialog {
    public static DialogTree createQuestDialog() {
        return new DialogTreeBuilder()
            .addNode("greeting")
                .text("Greetings, adventurer!")
                .addResponse("Do you have any work?")
                    .condition(ctx -> ctx.getNPC().hasQuestsAvailable(ctx.getPlayer()))
                    .goToNode("quest_available")
                    .build()
                .addResponse("About my quest...")
                    .condition(ctx -> ctx.getPlayer().hasActiveQuest("npc_quest"))
                    .goToNode("quest_progress")
                    .build()
                .addResponse("I've completed your task")
                    .condition(ctx -> ctx.getPlayer().canCompleteQuest("npc_quest"))
                    .goToNode("quest_complete")
                    .build()
                .addResponse("Farewell")
                    .endDialog()
                    .build()
                .build()
            
            .addNode("quest_available")
                .text("I need someone to gather herbs from the forest.")
                .showQuest("npc_quest")
                .addResponse("I'll do it")
                    .acceptQuest("npc_quest")
                    .goToNode("quest_accepted")
                    .build()
                .addResponse("Not right now")
                    .goToNode("greeting")
                    .build()
                .build()
            
            .addNode("quest_accepted")
                .text("Excellent! Return when you have the herbs.")
                .endDialog()
                .build()
            
            .addNode("quest_progress")
                .text(ctx -> {
                    Quest quest = ctx.getQuest("npc_quest");
                    int progress = quest.getProgress(ctx.getPlayer());
                    int total = quest.getTotalRequired();
                    return "You have collected " + progress + " out of " + total + " herbs.";
                })
                .addResponse("I'll keep looking")
                    .endDialog()
                    .build()
                .build()
            
            .addNode("quest_complete")
                .text("Wonderful! Here is your reward.")
                .completeQuest("npc_quest")
                .giveRewards()
                .addResponse("Thank you")
                    .endDialog()
                    .build()
                .build()
            
            .build();
    }
}
```

### Dynamic Dialog

```java
public class DynamicDialog {
    public static DialogNode createDynamicNode(NPC npc, Player player) {
        List<Quest> availableQuests = npc.getAvailableQuests(player);
        
        DialogNodeBuilder builder = new DialogNodeBuilder("quest_list")
            .text("I have several tasks available:");
        
        for (Quest quest : availableQuests) {
            builder.addResponse(quest.getName())
                .action(ctx -> ctx.showQuestDetails(quest))
                .goToNode("quest_details_" + quest.getId())
                .build();
        }
        
        builder.addResponse("None for now")
            .endDialog()
            .build();
        
        return builder.build();
    }
}
```

---

## Quest Integration

### Quest Giver Implementation

```java
public class QuestGiverNPC {
    public static NPC createQuestGiver() {
        NPC npc = new NPCBuilder("quest_giver")
            .withName("Quest Master")
            .withType(NPCType.QUEST_GIVER)
            .build();
        
        // Register quests
        npc.addQuest("beginner_quest_1");
        npc.addQuest("beginner_quest_2");
        npc.addQuest("advanced_quest_1");
        
        // Quest availability conditions
        npc.setQuestAvailability("advanced_quest_1", player -> 
            player.getLevel() >= 10 && 
            player.hasCompletedQuest("beginner_quest_2")
        );
        
        // Configure quest dialog
        npc.setDialogTree(createQuestGiverDialog());
        
        // Quest completion handler
        npc.onQuestComplete((player, quest) -> {
            player.sendMessage("§a" + npc.getName() + ": Well done!");
            
            // Unlock next quest
            String nextQuest = quest.getNextQuestId();
            if (nextQuest != null) {
                npc.unlockQuest(player, nextQuest);
            }
        });
        
        return npc;
    }
    
    private static DialogTree createQuestGiverDialog() {
        return new DialogTreeBuilder()
            .addNode("greeting")
                .text("Seeking adventure?")
                .addResponse("Show me available quests")
                    .action(ctx -> ctx.getNPC().showQuestList(ctx.getPlayer()))
                    .build()
                .addResponse("I've completed a quest")
                    .condition(ctx -> ctx.getNPC().hasCompletableQuests(ctx.getPlayer()))
                    .action(ctx -> ctx.getNPC().showCompletableQuests(ctx.getPlayer()))
                    .build()
                .addResponse("Goodbye")
                    .endDialog()
                    .build()
                .build()
            .build();
    }
}
```

### Quest Chain Integration

```java
public class QuestChainNPC {
    public static void setupQuestChain(NPC npc) {
        // Define quest chain
        QuestChain chain = new QuestChainBuilder("epic_chain")
            .addQuest("chapter_1")
            .addQuest("chapter_2")
            .addQuest("chapter_3")
            .build();
        
        npc.addQuestChain(chain);
        
        // Track chain progress
        npc.onQuestComplete((player, quest) -> {
            if (chain.contains(quest)) {
                int progress = chain.getProgress(player);
                int total = chain.getQuestCount();
                
                player.sendMessage(String.format(
                    "Quest chain progress: %d/%d", progress, total
                ));
                
                if (chain.isComplete(player)) {
                    giveChainReward(player);
                }
            }
        });
    }
    
    private static void giveChainReward(Player player) {
        player.sendMessage("§6You've completed the epic quest chain!");
        // Give special rewards
    }
}
```

### Multi-NPC Quests

```java
public class MultiNPCQuest {
    public static void setupMultiNPCQuest() {
        // Quest that involves multiple NPCs
        Quest quest = new QuestBuilder("multi_npc_quest")
            .withName("The Investigation")
            .addObjective(ObjectiveType.TALK_TO_NPC)
                .npcId("witness_1")
                .build()
            .addObjective(ObjectiveType.TALK_TO_NPC)
                .npcId("witness_2")
                .build()
            .addObjective(ObjectiveType.TALK_TO_NPC)
                .npcId("suspect")
                .build()
            .build();
        
        // Configure NPCs
        NPC witness1 = getNPC("witness_1");
        witness1.addQuestDialog("multi_npc_quest", 
            "I saw a suspicious figure near the market!");
        
        NPC witness2 = getNPC("witness_2");
        witness2.addQuestDialog("multi_npc_quest",
            "The thief wore a red cloak!");
        
        NPC suspect = getNPC("suspect");
        suspect.addQuestDialog("multi_npc_quest",
            "It wasn't me, I swear!");
    }
}
```

---

## NPC Behaviors

### Basic Behaviors

```java
public enum NPCBehavior {
    IDLE,       // Stand still
    WANDER,     // Random walking
    PATROL,     // Follow patrol path
    FOLLOW,     // Follow player
    FLEE,       // Run away from threats
    GUARD,      // Protect area
    ATTACK,     // Combat mode
    CUSTOM      // Custom behavior
}
```

### Idle Behavior

```java
public class IdleBehavior implements NPCBehaviorHandler {
    @Override
    public void execute(NPC npc) {
        // Look at nearby players
        Player nearestPlayer = npc.getNearestPlayer(10);
        if (nearestPlayer != null) {
            npc.lookAt(nearestPlayer);
        }
        
        // Occasional animations
        if (Math.random() < 0.01) {
            npc.playAnimation("idle_scratch");
        }
    }
}
```

### Wander Behavior

```java
public class WanderBehavior implements NPCBehaviorHandler {
    private Location homeLocation;
    private double wanderRadius;
    private Location targetLocation;
    private long lastWanderTime;
    
    public WanderBehavior(Location home, double radius) {
        this.homeLocation = home;
        this.wanderRadius = radius;
    }
    
    @Override
    public void execute(NPC npc) {
        long now = System.currentTimeMillis();
        
        // Find new target every 5-10 seconds
        if (targetLocation == null || 
            npc.hasReachedLocation(targetLocation) ||
            now - lastWanderTime > 5000 + Math.random() * 5000) {
            
            targetLocation = findRandomLocation();
            npc.navigateTo(targetLocation);
            lastWanderTime = now;
        }
    }
    
    private Location findRandomLocation() {
        double angle = Math.random() * 2 * Math.PI;
        double distance = Math.random() * wanderRadius;
        
        double x = homeLocation.getX() + Math.cos(angle) * distance;
        double z = homeLocation.getZ() + Math.sin(angle) * distance;
        
        return new Location(homeLocation.getWorld(), x, homeLocation.getY(), z);
    }
}
```

### Patrol Behavior

```java
public class PatrolBehavior implements NPCBehaviorHandler {
    private List<Location> patrolPoints;
    private int currentPoint = 0;
    private boolean reverse = false;
    
    public PatrolBehavior(List<Location> points) {
        this.patrolPoints = points;
    }
    
    @Override
    public void execute(NPC npc) {
        Location target = patrolPoints.get(currentPoint);
        
        if (npc.hasReachedLocation(target)) {
            // Wait at patrol point
            npc.waitFor(Duration.ofSeconds(2));
            
            // Move to next point
            if (reverse) {
                currentPoint--;
                if (currentPoint < 0) {
                    currentPoint = 1;
                    reverse = false;
                }
            } else {
                currentPoint++;
                if (currentPoint >= patrolPoints.size()) {
                    currentPoint = patrolPoints.size() - 2;
                    reverse = true;
                }
            }
        } else {
            npc.navigateTo(target);
        }
    }
}
```

### Follow Behavior

```java
public class FollowBehavior implements NPCBehaviorHandler {
    private Player target;
    private double followDistance = 3.0;
    private double maxDistance = 20.0;
    
    public FollowBehavior(Player target) {
        this.target = target;
    }
    
    @Override
    public void execute(NPC npc) {
        if (!target.isOnline()) {
            npc.setBehavior(NPCBehavior.IDLE);
            return;
        }
        
        double distance = npc.getLocation().distance(target.getLocation());
        
        if (distance > maxDistance) {
            // Teleport if too far
            npc.teleport(target.getLocation());
        } else if (distance > followDistance) {
            // Navigate to player
            npc.navigateTo(target.getLocation());
        } else {
            // Close enough, look at player
            npc.lookAt(target);
            npc.stopMoving();
        }
    }
}
```

### Guard Behavior

```java
public class GuardBehavior implements NPCBehaviorHandler {
    private Location guardPost;
    private double guardRadius;
    
    public GuardBehavior(Location post, double radius) {
        this.guardPost = post;
        this.guardRadius = radius;
    }
    
    @Override
    public void execute(NPC npc) {
        // Return to post if too far
        if (npc.getLocation().distance(guardPost) > guardRadius) {
            npc.navigateTo(guardPost);
            return;
        }
        
        // Look for threats
        Entity threat = findNearestThreat(npc);
        if (threat != null) {
            npc.lookAt(threat);
            
            if (threat.getLocation().distance(npc.getLocation()) < 10) {
                npc.attackEntity(threat);
            }
        } else {
            // Scan area
            npc.rotateSlowly();
        }
    }
    
    private Entity findNearestThreat(NPC npc) {
        return npc.getNearbyEntities(guardRadius).stream()
            .filter(e -> e instanceof Monster)
            .min(Comparator.comparingDouble(e -> 
                e.getLocation().distance(npc.getLocation())))
            .orElse(null);
    }
}
```

---

## Pathfinding and Movement

### Basic Pathfinding

```java
public class NPCPathfinding {
    public void navigateToLocation(NPC npc, Location target) {
        Path path = pathfinder.findPath(npc.getLocation(), target);
        
        if (path != null && path.isValid()) {
            npc.followPath(path);
        } else {
            // Fallback: direct navigation
            npc.moveTowards(target);
        }
    }
    
    public void navigateToPlayer(NPC npc, Player player) {
        navigateToLocation(npc, player.getLocation());
    }
    
    public void navigateToNPC(NPC npc, NPC target) {
        navigateToLocation(npc, target.getLocation());
    }
}
```

### Advanced Pathfinding

```java
public class AdvancedPathfinding {
    public Path findOptimalPath(NPC npc, Location target) {
        PathfinderOptions options = new PathfinderOptions()
            .avoidWater(npc.avoidsWater())
            .avoidFire(true)
            .canOpenDoors(npc.canOpenDoors())
            .canClimb(npc.canClimb())
            .maxFallDistance(npc.getMaxFallDistance())
            .preferredSpeed(npc.getMovementSpeed());
        
        return pathfinder.findPath(npc.getLocation(), target, options);
    }
    
    public void navigateAround(NPC npc, Location obstacle, Location goal) {
        Path path = pathfinder.findPathAvoiding(
            npc.getLocation(), 
            goal, 
            Collections.singletonList(obstacle),
            5.0  // Avoidance radius
        );
        
        npc.followPath(path);
    }
}
```

### Movement Patterns

```java
public class MovementPatterns {
    // Circular patrol
    public void circularPatrol(NPC npc, Location center, double radius) {
        List<Location> circle = new ArrayList<>();
        int points = 16;
        
        for (int i = 0; i < points; i++) {
            double angle = (2 * Math.PI * i) / points;
            double x = center.getX() + radius * Math.cos(angle);
            double z = center.getZ() + radius * Math.sin(angle);
            circle.add(new Location(center.getWorld(), x, center.getY(), z));
        }
        
        npc.setBehavior(new PatrolBehavior(circle));
    }
    
    // Figure-8 pattern
    public void figure8Pattern(NPC npc, Location center) {
        List<Location> pattern = new ArrayList<>();
        int points = 32;
        double radius = 5.0;
        
        for (int i = 0; i < points; i++) {
            double t = (2 * Math.PI * i) / points;
            double x = center.getX() + radius * Math.sin(t);
            double z = center.getZ() + radius * Math.sin(t) * Math.cos(t);
            pattern.add(new Location(center.getWorld(), x, center.getY(), z));
        }
        
        npc.setBehavior(new PatrolBehavior(pattern));
    }
}
```

---

## NPC Interactions

### Custom Interactions

```java
public class CustomNPCInteractions {
    public static void setupInteractions(NPC npc) {
        // Right-click interaction
        npc.onInteract(player -> {
            if (player.isSneaking()) {
                // Sneaking + right-click: special action
                npc.performSpecialAction(player);
            } else {
                // Normal right-click: dialog
                npc.startDialog(player);
            }
        });
        
        // Item interaction
        npc.onItemUse((player, item) -> {
            if (item.getType().equals("quest_item")) {
                npc.acceptQuestItem(player, item);
                return true;
            }
            return false;
        });
        
        // Attack interaction
        npc.onAttacked((player, damage) -> {
            if (npc.isFriendly()) {
                npc.becomeHostile(player);
                npc.callForHelp();
            }
        });
    }
}
```

### Reputation System

```java
public class NPCReputationSystem {
    public void setupReputation(NPC npc) {
        npc.setReputationRequired("friendly", 0);
        npc.setReputationRequired("honored", 500);
        npc.setReputationRequired("exalted", 1000);
        
        // Reputation-based dialog
        npc.addDialogVariant("greeting", player -> {
            int rep = player.getReputation(npc.getFaction());
            
            if (rep >= 1000) {
                return "Greetings, exalted one!";
            } else if (rep >= 500) {
                return "Welcome, honored friend!";
            } else if (rep >= 0) {
                return "Hello, traveler.";
            } else {
                return "What do you want?";
            }
        });
        
        // Reputation-based discounts
        if (npc.isMerchant()) {
            npc.setPriceModifier(player -> {
                int rep = player.getReputation(npc.getFaction());
                return 1.0 - (rep / 2000.0); // Up to 50% discount
            });
        }
    }
}
```

---

## Advanced Features

### NPC Emotions and States

```java
public class NPCEmotions {
    public enum Emotion {
        HAPPY, SAD, ANGRY, FEARFUL, NEUTRAL
    }
    
    public void setupEmotions(NPC npc) {
        NPCEmotionSystem emotions = new NPCEmotionSystem(npc);
        
        emotions.onEmotionChange((oldEmotion, newEmotion) -> {
            switch (newEmotion) {
                case HAPPY:
                    npc.playAnimation("cheer");
                    npc.setDialogTone("cheerful");
                    break;
                case ANGRY:
                    npc.playAnimation("angry_gesture");
                    npc.setDialogTone("aggressive");
                    break;
                case FEARFUL:
                    npc.setBehavior(NPCBehavior.FLEE);
                    break;
            }
        });
        
        npc.setEmotionSystem(emotions);
    }
}
```

### NPC Schedules

```java
public class NPCSchedule {
    public void setupDailySchedule(NPC npc) {
        Schedule schedule = new ScheduleBuilder()
            .addActivity(TimeRange.from(6, 0).to(8, 0))
                .behavior(NPCBehavior.WANDER)
                .location("home")
                .build()
            
            .addActivity(TimeRange.from(8, 0).to(12, 0))
                .behavior(NPCBehavior.IDLE)
                .location("shop")
                .enableTrading(true)
                .build()
            
            .addActivity(TimeRange.from(12, 0).to(13, 0))
                .behavior(NPCBehavior.WANDER)
                .location("tavern")
                .build()
            
            .addActivity(TimeRange.from(13, 0).to(18, 0))
                .behavior(NPCBehavior.IDLE)
                .location("shop")
                .enableTrading(true)
                .build()
            
            .addActivity(TimeRange.from(18, 0).to(22, 0))
                .behavior(NPCBehavior.WANDER)
                .location("home")
                .build()
            
            .addActivity(TimeRange.from(22, 0).to(6, 0))
                .behavior(NPCBehavior.IDLE)
                .location("home")
                .invisible(true)
                .build()
            
            .build();
        
        npc.setSchedule(schedule);
    }
}
```

---

## Best Practices

### DO:
- ✅ Use meaningful NPC IDs and names
- ✅ Test dialog flows thoroughly
- ✅ Implement fallback behaviors
- ✅ Cache pathfinding results
- ✅ Clean up despawned NPCs
- ✅ Use events for NPC interactions
- ✅ Implement proper error handling

### DON'T:
- ❌ Spawn too many NPCs in one area
- ❌ Use complex pathfinding every tick
- ❌ Forget to despawn NPCs
- ❌ Make dialogs too long
- ❌ Ignore performance implications
- ❌ Hard-code NPC data

---

## Troubleshooting

### NPC Not Spawning

```java
public void diagnoseSpawnIssue(String npcId) {
    NPC npc = npcManager.getNPC(npcId);
    
    if (npc == null) {
        logger.error("NPC not registered: " + npcId);
        return;
    }
    
    Location loc = npc.getSpawnLocation();
    if (!loc.getWorld().isLoaded()) {
        logger.error("World not loaded: " + loc.getWorld().getName());
    }
    
    if (!loc.getChunk().isLoaded()) {
        logger.error("Chunk not loaded at: " + loc);
    }
}
```

### Pathfinding Issues

```java
public void debugPathfinding(NPC npc, Location target) {
    Path path = pathfinder.findPath(npc.getLocation(), target);
    
    if (path == null) {
        logger.warn("No path found from {} to {}", npc.getLocation(), target);
    } else {
        logger.info("Path length: {} blocks", path.getLength());
        logger.info("Path points: {}", path.getPoints().size());
    }
}
```

For more information, see:
- [Quest Development Guide](quest-development.md)
- [UI Development Guide](ui-development.md)
- [API Reference](../api/framework-npc.md)
