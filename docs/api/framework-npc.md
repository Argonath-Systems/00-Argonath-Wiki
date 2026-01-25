# NPC Framework API Reference

**Package:** `com.argonathsystems.framework.npc`  
**Module ID:** `npc`  
**Version:** 1.0.0  
**Dependencies:** `core`, `config`, `storage`

## Overview

The NPC Framework provides a comprehensive system for creating interactive non-player characters with dialog trees, behaviors, quest integration, and custom interactions. Features include dialog branching, conditions, animations, and pathfinding.

## Table of Contents

- [NPC](#npc)
- [NPCBuilder](#npcbuilder)
- [NPCRegistry](#npcregistry)
- [NPCManager](#npcmanager)
- [DialogNode](#dialognode)
- [DialogChoice](#dialogchoice)
- [DialogTree](#dialogtree)
- [NPCBehavior](#npcbehavior)
- [InteractionHandler](#interactionhandler)
- [NPC Events](#npc-events)

---

## NPC

Represents a non-player character with properties, behaviors, and dialog.

### Class Declaration

```java
public interface NPC {
    String getId();
    String getName();
    EntityType getEntityType();
    Location getLocation();
    void setLocation(Location location);
    String getSkin();
    void setSkin(String skin);
    DialogTree getDialogTree();
    void setDialogTree(DialogTree dialogTree);
    List<NPCBehavior> getBehaviors();
    void addBehavior(NPCBehavior behavior);
    void removeBehavior(NPCBehavior behavior);
    Map<String, Object> getMetadata();
    void setMetadata(String key, Object value);
    Object getMetadata(String key);
    boolean isSpawned();
    void spawn();
    void despawn();
    void lookAt(Location location);
    void lookAt(Entity entity);
    void playAnimation(NPCAnimation animation);
}
```

### Methods

#### `getId()`

Returns the unique NPC identifier.

```java
NPC npc = npcManager.getNPC("blacksmith_john");
String id = npc.getId(); // "blacksmith_john"
```

**Returns:** Unique NPC ID  
**Immutability:** Immutable after creation

---

#### `getName()`

Returns the display name shown above the NPC.

```java
String name = npc.getName(); // "§6John the Blacksmith"
```

**Returns:** Display name with color code support  
**Visibility:** Shown in entity nameplate

---

#### `getEntityType()`

Returns the entity type used for the NPC.

```java
EntityType type = npc.getEntityType();
// PLAYER, VILLAGER, ZOMBIE, SKELETON, etc.
```

**Returns:** NPC entity type  
**Default:** `EntityType.PLAYER`

---

#### `setLocation(Location location)`

Moves the NPC to a new location.

```java
Location newLocation = new Location(world, 100, 64, 200);
npc.setLocation(newLocation);
```

**Parameters:**
- `location` - New location for the NPC

**Side Effects:** Teleports NPC if spawned  
**Thread Safety:** Thread-safe

---

#### `getDialogTree()`

Returns the NPC's dialog tree.

```java
DialogTree dialog = npc.getDialogTree();
DialogNode root = dialog.getRootNode();
```

**Returns:** `DialogTree` instance or `null` if no dialog  
**See Also:** [DialogTree](#dialogtree)

---

#### `spawn()`

Spawns the NPC in the world.

```java
npc.spawn();
// NPC now visible and interactable
```

**Events:** Fires `NPCSpawnEvent`  
**Side Effects:** Creates entity in world  
**Thread Safety:** Must be called on main thread

---

#### `despawn()`

Removes the NPC from the world.

```java
npc.despawn();
// NPC no longer visible
```

**Events:** Fires `NPCDespawnEvent`  
**Side Effects:** Removes entity from world  
**Preservation:** Dialog state and metadata preserved

---

#### `lookAt(Entity entity)`

Makes the NPC look at an entity.

```java
eventBus.subscribe(PlayerInteractNPCEvent.class, event -> {
    NPC npc = event.getNPC();
    Player player = event.getPlayer();
    npc.lookAt(player);
});
```

**Parameters:**
- `entity` - Entity to look at

**Animation:** Smoothly rotates head and body  
**Thread Safety:** Thread-safe

---

#### `playAnimation(NPCAnimation animation)`

Plays an animation.

```java
npc.playAnimation(NPCAnimation.WAVE);
// NPC waves at player
```

**Parameters:**
- `animation` - Animation to play

**Animations:** WAVE, NOD, SHAKE_HEAD, POINT, ATTACK, CROUCH, etc.  
**Duration:** Varies by animation type

---

### Example Usage

```java
public class QuestGiverNPC {
    public void createQuestGiver(Location location) {
        NPC npc = NPC.builder()
            .id("quest_giver_aldric")
            .name("§6Aldric the Wise")
            .entityType(EntityType.PLAYER)
            .location(location)
            .skin("ewogICJ0aW1lc3RhbXAiIDogMTY0...")  // Base64 skin data
            .dialogTree(createDialogTree())
            .behavior(new LookAtPlayerBehavior(5.0))
            .behavior(new WaveOnApproachBehavior())
            .metadata("quest_giver", true)
            .metadata("available_quests", Arrays.asList("main_quest_1", "side_quest_3"))
            .build();
        
        npcManager.registerNPC(npc);
        npc.spawn();
    }
    
    private DialogTree createDialogTree() {
        return DialogTree.builder()
            .rootNode(
                DialogNode.builder()
                    .text("Greetings, traveler! I have tasks for those brave enough.")
                    .choice(
                        DialogChoice.builder()
                            .text("What quests do you have?")
                            .action(DialogAction.SHOW_QUEST_LIST)
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Tell me about yourself.")
                            .nextNode("about")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Goodbye.")
                            .action(DialogAction.END_CONVERSATION)
                            .build()
                    )
                    .build()
            )
            .node("about",
                DialogNode.builder()
                    .text("I am Aldric, keeper of ancient knowledge and quests.")
                    .choice(
                        DialogChoice.builder()
                            .text("Back.")
                            .nextNode("root")
                            .build()
                    )
                    .build()
            )
            .build();
    }
}
```

---

## NPCBuilder

Fluent builder for creating NPC instances.

### Class Declaration

```java
public interface NPCBuilder {
    NPCBuilder id(String id);
    NPCBuilder name(String name);
    NPCBuilder entityType(EntityType type);
    NPCBuilder location(Location location);
    NPCBuilder skin(String skinData);
    NPCBuilder dialogTree(DialogTree dialogTree);
    NPCBuilder behavior(NPCBehavior behavior);
    NPCBuilder behaviors(NPCBehavior... behaviors);
    NPCBuilder metadata(String key, Object value);
    NPCBuilder interactionHandler(InteractionHandler handler);
    NPC build();
}
```

### Methods

#### `id(String id)`

Sets the unique NPC identifier.

```java
NPCBuilder builder = NPC.builder()
    .id("blacksmith_john");
```

**Parameters:**
- `id` - Unique NPC ID (required)

**Returns:** Builder for chaining  
**Validation:** Must be unique, alphanumeric with underscores

---

#### `name(String name)`

Sets the NPC display name.

```java
builder.name("§6John the Blacksmith");
```

**Parameters:**
- `name` - Display name (supports color codes)

**Returns:** Builder for chaining

---

#### `entityType(EntityType type)`

Sets the entity type for the NPC.

```java
builder.entityType(EntityType.VILLAGER);
```

**Parameters:**
- `type` - Entity type

**Returns:** Builder for chaining  
**Default:** `EntityType.PLAYER`

---

#### `location(Location location)`

Sets the spawn location.

```java
Location spawn = new Location(world, 100, 64, 200, 90, 0);
builder.location(spawn);
```

**Parameters:**
- `location` - Spawn location with yaw/pitch

**Returns:** Builder for chaining  
**Required:** Yes

---

#### `skin(String skinData)`

Sets the skin for player NPCs.

```java
builder.skin("ewogICJ0aW1lc3RhbXAiIDogMTY0...");
```

**Parameters:**
- `skinData` - Base64-encoded skin texture data

**Returns:** Builder for chaining  
**Applies To:** `EntityType.PLAYER` only

---

#### `behavior(NPCBehavior behavior)`

Adds a behavior to the NPC.

```java
builder.behavior(new LookAtPlayerBehavior(5.0))
       .behavior(new RandomWalkBehavior())
       .behavior(new GreetPlayerBehavior());
```

**Parameters:**
- `behavior` - Behavior instance

**Returns:** Builder for chaining  
**See Also:** [NPCBehavior](#npcbehavior)

---

### Complete Example

```java
public class NPCFactory {
    public static NPC createVillageGuard(World world) {
        return NPC.builder()
            .id("village_guard_1")
            .name("§4Guard Captain")
            .entityType(EntityType.PLAYER)
            .location(new Location(world, 100, 64, 100, 180, 0))
            .skin(SkinLibrary.GUARD_SKIN)
            
            // Dialog
            .dialogTree(
                DialogTree.builder()
                    .rootNode(
                        DialogNode.builder()
                            .text("Halt! State your business.")
                            .condition(player -> !player.hasPermission("village.resident"))
                            .choice(
                                DialogChoice.builder()
                                    .text("I'm just passing through.")
                                    .nextNode("passing_through")
                                    .build()
                            )
                            .build()
                    )
                    .build()
            )
            
            // Behaviors
            .behavior(new LookAtPlayerBehavior(8.0))
            .behavior(new PatrolBehavior(Arrays.asList(
                new Location(world, 100, 64, 100),
                new Location(world, 120, 64, 100),
                new Location(world, 120, 64, 120),
                new Location(world, 100, 64, 120)
            )))
            
            // Metadata
            .metadata("guard", true)
            .metadata("patrol_route", "village_perimeter")
            
            .build();
    }
}
```

---

## NPCRegistry

Central registry for NPC definitions.

### Class Declaration

```java
public interface NPCRegistry {
    void registerNPC(NPC npc);
    void unregisterNPC(String npcId);
    NPC getNPC(String npcId);
    Collection<NPC> getAllNPCs();
    Collection<NPC> getNPCsInWorld(World world);
    Collection<NPC> getNPCsNearLocation(Location location, double radius);
    boolean isRegistered(String npcId);
    void spawnAll();
    void despawnAll();
}
```

### Methods

#### `registerNPC(NPC npc)`

Registers an NPC.

```java
NPCRegistry registry = npcManager.getRegistry();
NPC npc = createBlacksmith();
registry.registerNPC(npc);
```

**Parameters:**
- `npc` - NPC to register

**Throws:** `IllegalArgumentException` if NPC ID already registered  
**Thread Safety:** Thread-safe

---

#### `getNPC(String npcId)`

Retrieves an NPC by ID.

```java
NPC blacksmith = registry.getNPC("blacksmith_john");
if (blacksmith == null) {
    logger.warn("NPC not found: blacksmith_john");
}
```

**Parameters:**
- `npcId` - NPC identifier

**Returns:** NPC instance or `null` if not found  
**Thread Safety:** Thread-safe

---

#### `getNPCsNearLocation(Location location, double radius)`

Finds NPCs near a location.

```java
Location center = player.getLocation();
Collection<NPC> nearbyNPCs = registry.getNPCsNearLocation(center, 10.0);

for (NPC npc : nearbyNPCs) {
    npc.lookAt(player);
}
```

**Parameters:**
- `location` - Center location
- `radius` - Search radius in blocks

**Returns:** Collection of NPCs within radius  
**Performance:** Uses spatial indexing for efficiency

---

#### `spawnAll()`

Spawns all registered NPCs.

```java
// On server start
registry.spawnAll();
logger.info("Spawned {} NPCs", registry.getAllNPCs().size());
```

**Events:** Fires `NPCSpawnEvent` for each NPC  
**Thread Safety:** Must be called on main thread

---

### Example Usage

```java
public class NPCLoader {
    private final NPCRegistry registry;
    
    public void loadNPCs(Path npcDir) {
        logger.info("Loading NPCs from {}", npcDir);
        
        try (Stream<Path> paths = Files.walk(npcDir)) {
            paths.filter(p -> p.toString().endsWith(".yml"))
                 .forEach(path -> {
                     try {
                         NPC npc = parseNPCFile(path);
                         registry.registerNPC(npc);
                         logger.debug("Loaded NPC: {}", npc.getId());
                     } catch (Exception e) {
                         logger.error("Failed to load NPC from {}", path, e);
                     }
                 });
        } catch (IOException e) {
            logger.error("Failed to load NPCs", e);
        }
        
        // Spawn all loaded NPCs
        registry.spawnAll();
        logger.info("Spawned {} NPCs", registry.getAllNPCs().size());
    }
}
```

---

## NPCManager

High-level NPC management interface.

### Class Declaration

```java
public interface NPCManager {
    NPCRegistry getRegistry();
    void handleInteraction(Player player, NPC npc);
    void startDialog(Player player, NPC npc);
    void endDialog(Player player);
    DialogState getDialogState(Player player);
    void selectChoice(Player player, int choiceIndex);
    void updateBehaviors();
}
```

### Methods

#### `handleInteraction(Player player, NPC npc)`

Handles player-NPC interaction.

```java
npcManager.handleInteraction(player, npc);
// Starts dialog or triggers custom interaction
```

**Parameters:**
- `player` - Interacting player
- `npc` - NPC being interacted with

**Events:** Fires `PlayerInteractNPCEvent`  
**Side Effects:** May start dialog

---

#### `startDialog(Player player, NPC npc)`

Starts a dialog between player and NPC.

```java
npcManager.startDialog(player, npc);
```

**Parameters:**
- `player` - Player
- `npc` - NPC

**Events:** Fires `DialogStartEvent`  
**Side Effects:** Opens dialog interface, stores dialog state

---

#### `getDialogState(Player player)`

Gets the current dialog state for a player.

```java
DialogState state = npcManager.getDialogState(player);
if (state != null) {
    DialogNode currentNode = state.getCurrentNode();
    NPC npc = state.getNPC();
    logger.info("Player in dialog with {} at node {}", 
                npc.getName(), currentNode.getId());
}
```

**Parameters:**
- `player` - Player

**Returns:** `DialogState` or `null` if not in dialog

---

#### `selectChoice(Player player, int choiceIndex)`

Selects a dialog choice.

```java
// Player clicked choice #2
npcManager.selectChoice(player, 1);
```

**Parameters:**
- `player` - Player
- `choiceIndex` - Zero-based choice index

**Events:** Fires `DialogChoiceEvent`  
**Side Effects:** Advances dialog, may execute actions

---

### Example Usage

```java
public class NPCInteractionListener {
    private final NPCManager npcManager;
    
    public void register() {
        eventBus.subscribe(PlayerInteractEntityEvent.class, this::onEntityInteract);
        eventBus.subscribe(DialogChoiceEvent.class, this::onDialogChoice);
    }
    
    private void onEntityInteract(PlayerInteractEntityEvent event) {
        Entity entity = event.getEntity();
        Player player = event.getPlayer();
        
        // Check if entity is an NPC
        String npcId = entity.getMetadata("npc_id");
        if (npcId != null) {
            NPC npc = npcManager.getRegistry().getNPC(npcId);
            if (npc != null) {
                event.setCancelled(true);
                npcManager.handleInteraction(player, npc);
            }
        }
    }
    
    private void onDialogChoice(DialogChoiceEvent event) {
        Player player = event.getPlayer();
        DialogChoice choice = event.getChoice();
        
        logger.info("Player {} selected: {}", 
                   player.getName(), choice.getText());
        
        // Execute custom actions based on choice
        if (choice.getAction() == DialogAction.SHOW_QUEST_LIST) {
            showAvailableQuests(player, event.getNPC());
        }
    }
}
```

---

## DialogNode

Represents a node in a dialog tree.

### Class Declaration

```java
public interface DialogNode {
    String getId();
    String getText();
    List<DialogChoice> getChoices();
    Predicate<Player> getCondition();
    Consumer<Player> getAction();
    boolean isEndNode();
    
    static DialogNodeBuilder builder() {
        return new DialogNodeBuilderImpl();
    }
}
```

### Methods

#### `getText()`

Returns the dialog text displayed to the player.

```java
DialogNode node = dialogTree.getCurrentNode();
String text = node.getText();
player.sendMessage("§e[NPC] §f" + text);
```

**Returns:** Dialog text with color code support  
**Placeholders:** Supports `{player}`, `{npc}`, etc.

---

#### `getChoices()`

Returns available dialog choices.

```java
List<DialogChoice> choices = node.getChoices();
for (int i = 0; i < choices.size(); i++) {
    DialogChoice choice = choices.get(i);
    player.sendMessage(String.format("§7[%d] §f%s", i + 1, choice.getText()));
}
```

**Returns:** Immutable list of dialog choices  
**Filtering:** Choices with unsatisfied conditions are excluded

---

#### `getCondition()`

Returns the condition that must be met to show this node.

```java
Predicate<Player> condition = node.getCondition();
if (condition.test(player)) {
    // Show dialog
}
```

**Returns:** Condition predicate or `null` if no condition  
**Default:** Always returns `true` if no condition set

---

### Example Usage

```java
DialogNode questNode = DialogNode.builder()
    .id("quest_offer")
    .text("I have a quest for you, {player}. Are you interested?")
    .condition(player -> {
        // Only show if player doesn't have active quest
        QuestManager qm = getQuestManager();
        return !qm.isActive(player, "main_quest_1");
    })
    .choice(
        DialogChoice.builder()
            .text("Yes, I'll help you!")
            .action(player -> {
                QuestManager qm = getQuestManager();
                qm.startQuest(player, "main_quest_1");
            })
            .nextNode("quest_accepted")
            .build()
    )
    .choice(
        DialogChoice.builder()
            .text("Not right now.")
            .nextNode("root")
            .build()
    )
    .build();
```

---

## DialogChoice

Represents a dialog choice/option.

### Class Declaration

```java
public interface DialogChoice {
    String getText();
    String getNextNodeId();
    Predicate<Player> getCondition();
    Consumer<Player> getAction();
    DialogAction getAction();
    
    static DialogChoiceBuilder builder() {
        return new DialogChoiceBuilderImpl();
    }
}
```

### Methods

#### `getText()`

Returns the choice text displayed to player.

```java
String choiceText = choice.getText();
// "Tell me about the quest."
```

**Returns:** Choice display text

---

#### `getNextNodeId()`

Returns the ID of the node to navigate to.

```java
String nextNode = choice.getNextNodeId();
// Navigate to next node after choice selected
```

**Returns:** Next node ID or `null` to end dialog

---

#### `getCondition()`

Returns the condition for showing this choice.

```java
DialogChoice premiumChoice = DialogChoice.builder()
    .text("I'd like to see premium quests.")
    .condition(player -> player.hasPermission("quests.premium"))
    .nextNode("premium_quests")
    .build();
```

**Returns:** Condition predicate or `null`  
**Behavior:** Choice hidden if condition fails

---

#### `getAction()`

Returns the action to execute when choice is selected.

```java
DialogChoice acceptQuest = DialogChoice.builder()
    .text("I accept the quest!")
    .action(player -> {
        questManager.startQuest(player, "main_quest_1");
        player.sendMessage("§aQuest started!");
    })
    .nextNode("quest_accepted")
    .build();
```

**Returns:** Action consumer or `null`  
**Execution:** Runs when choice is selected

---

### Dialog Actions

```java
public enum DialogAction {
    END_CONVERSATION,
    SHOW_QUEST_LIST,
    SHOW_SHOP,
    GIVE_ITEM,
    TELEPORT,
    CUSTOM
}
```

---

## DialogTree

Represents a complete dialog tree structure.

### Class Declaration

```java
public interface DialogTree {
    DialogNode getRootNode();
    DialogNode getNode(String nodeId);
    Map<String, DialogNode> getAllNodes();
    
    static DialogTreeBuilder builder() {
        return new DialogTreeBuilderImpl();
    }
}
```

### Example Usage

```java
public class DialogTreeFactory {
    public static DialogTree createQuestGiverDialog() {
        return DialogTree.builder()
            .rootNode(
                DialogNode.builder()
                    .id("root")
                    .text("Greetings, brave adventurer! How may I assist you?")
                    .choice(
                        DialogChoice.builder()
                            .text("What quests do you have?")
                            .nextNode("quest_list")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Tell me about this place.")
                            .nextNode("about_location")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Goodbye.")
                            .action(DialogAction.END_CONVERSATION)
                            .build()
                    )
                    .build()
            )
            .node("quest_list",
                DialogNode.builder()
                    .id("quest_list")
                    .text("I have several quests available:")
                    .choice(
                        DialogChoice.builder()
                            .text("Main Quest: The Lost Artifact")
                            .condition(player -> 
                                !questManager.isCompleted(player, "main_quest_1"))
                            .nextNode("quest_main_1")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Side Quest: Help the Farmers")
                            .condition(player ->
                                !questManager.isCompleted(player, "side_quest_1"))
                            .nextNode("quest_side_1")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Back.")
                            .nextNode("root")
                            .build()
                    )
                    .build()
            )
            .node("quest_main_1",
                DialogNode.builder()
                    .id("quest_main_1")
                    .text("The ancient artifact was lost in the mountain caves. Will you retrieve it?")
                    .choice(
                        DialogChoice.builder()
                            .text("I accept this quest!")
                            .action(player -> questManager.startQuest(player, "main_quest_1"))
                            .nextNode("quest_accepted")
                            .build()
                    )
                    .choice(
                        DialogChoice.builder()
                            .text("Not right now.")
                            .nextNode("quest_list")
                            .build()
                    )
                    .build()
            )
            .node("quest_accepted",
                DialogNode.builder()
                    .id("quest_accepted")
                    .text("Excellent! Good luck on your journey.")
                    .choice(
                        DialogChoice.builder()
                            .text("Thank you!")
                            .action(DialogAction.END_CONVERSATION)
                            .build()
                    )
                    .build()
            )
            .build();
    }
}
```

---

## NPCBehavior

Defines autonomous NPC behavior.

### Interface

```java
public interface NPCBehavior {
    void tick(NPC npc);
    boolean shouldExecute(NPC npc);
    void start(NPC npc);
    void stop(NPC npc);
    int getPriority();
}
```

### Built-in Behaviors

#### LookAtPlayerBehavior

Makes NPC look at nearby players.

```java
NPCBehavior lookAtPlayer = new LookAtPlayerBehavior(5.0); // 5 block radius
npc.addBehavior(lookAtPlayer);
```

#### PatrolBehavior

Makes NPC patrol between waypoints.

```java
List<Location> waypoints = Arrays.asList(
    new Location(world, 100, 64, 100),
    new Location(world, 120, 64, 100),
    new Location(world, 120, 64, 120)
);
NPCBehavior patrol = new PatrolBehavior(waypoints);
npc.addBehavior(patrol);
```

#### GreetPlayerBehavior

Makes NPC greet nearby players.

```java
NPCBehavior greet = new GreetPlayerBehavior(
    3.0,  // Radius
    "Hello, {player}!",
    Duration.ofMinutes(5)  // Cooldown
);
npc.addBehavior(greet);
```

### Custom Behavior Example

```java
public class QuestReminderBehavior implements NPCBehavior {
    private final String questId;
    private final double radius;
    
    public QuestReminderBehavior(String questId, double radius) {
        this.questId = questId;
        this.radius = radius;
    }
    
    @Override
    public void tick(NPC npc) {
        Collection<Player> nearbyPlayers = 
            npc.getLocation().getNearbyPlayers(radius);
        
        for (Player player : nearbyPlayers) {
            QuestManager qm = getQuestManager();
            if (qm.isActive(player, questId)) {
                QuestProgress progress = qm.getProgress(player, questId);
                if (progress.getCompletionPercentage() == 100) {
                    npc.lookAt(player);
                    player.sendMessage("§e[" + npc.getName() + "] §fI see you've completed the quest!");
                }
            }
        }
    }
    
    @Override
    public boolean shouldExecute(NPC npc) {
        return npc.isSpawned() && !npc.getLocation().getNearbyPlayers(radius).isEmpty();
    }
    
    @Override
    public int getPriority() {
        return 5;
    }
}
```

---

## InteractionHandler

Handles custom NPC interactions.

### Interface

```java
public interface InteractionHandler {
    void handleInteraction(Player player, NPC npc, InteractionType type);
}
```

### Interaction Types

```java
public enum InteractionType {
    RIGHT_CLICK,
    LEFT_CLICK,
    SHIFT_RIGHT_CLICK,
    SHIFT_LEFT_CLICK
}
```

### Example

```java
public class ShopKeeperInteraction implements InteractionHandler {
    @Override
    public void handleInteraction(Player player, NPC npc, InteractionType type) {
        if (type == InteractionType.RIGHT_CLICK) {
            openShop(player);
        } else if (type == InteractionType.SHIFT_RIGHT_CLICK) {
            showShopInfo(player);
        }
    }
    
    private void openShop(Player player) {
        // Open shop GUI
    }
}

// Register handler
NPC shopKeeper = NPC.builder()
    .id("shop_keeper")
    .name("§6Shop Keeper")
    .interactionHandler(new ShopKeeperInteraction())
    .build();
```

---

## NPC Events

### NPCSpawnEvent

Fired when an NPC spawns.

```java
public class NPCSpawnEvent extends Event implements Cancellable {
    private final NPC npc;
    private boolean cancelled;
    
    public NPC getNPC() { return npc; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(NPCSpawnEvent.class, event -> {
    NPC npc = event.getNPC();
    logger.info("NPC spawned: {} at {}", npc.getName(), npc.getLocation());
});
```

---

### PlayerInteractNPCEvent

Fired when a player interacts with an NPC.

```java
public class PlayerInteractNPCEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final InteractionType interactionType;
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public InteractionType getInteractionType() { return interactionType; }
    public boolean isCancelled() { return cancelled; }
    public void setCancelled(boolean cancelled) { this.cancelled = cancelled; }
}
```

**Example:**
```java
eventBus.subscribe(PlayerInteractNPCEvent.class, event -> {
    Player player = event.getPlayer();
    NPC npc = event.getNPC();
    
    // Custom interaction logic
    if (npc.getMetadata("quest_giver") == Boolean.TRUE) {
        showAvailableQuests(player, npc);
        event.setCancelled(true);  // Prevent default dialog
    }
});
```

---

### DialogStartEvent

Fired when dialog starts.

```java
public class DialogStartEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final DialogTree dialogTree;
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public DialogTree getDialogTree() { return dialogTree; }
}
```

---

### DialogChoiceEvent

Fired when player selects a dialog choice.

```java
public class DialogChoiceEvent extends Event implements Cancellable {
    private final Player player;
    private final NPC npc;
    private final DialogNode currentNode;
    private final DialogChoice choice;
    private boolean cancelled;
    
    public Player getPlayer() { return player; }
    public NPC getNPC() { return npc; }
    public DialogNode getCurrentNode() { return currentNode; }
    public DialogChoice getChoice() { return choice; }
}
```

---

## See Also

- [Platform Core API](platform-core.md) - Core platform services
- [Quest Framework API](framework-quest.md) - Quest integration
- [Event System API](events.md) - Complete event reference
- [UI Framework API](framework-ui.md) - Dialog UI components
