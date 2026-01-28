# Dialogue System Guide

**Version:** 1.1.0  
**Last Updated:** 2026-01-28  
**Status:** ✅ Implemented

---

## Overview

The Argonath Dialogue System provides a comprehensive, platform-agnostic framework for creating interactive NPC conversations. It supports branching dialogue trees, conditional responses, quest integration, and rich narrative experiences.

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Creating Dialogue Trees](#creating-dialogue-trees)
3. [Dialogue Nodes](#dialogue-nodes)
4. [Dialogue Options](#dialogue-options)
5. [Conditions and Actions](#conditions-and-actions)
6. [Quest Integration](#quest-integration)
7. [NPC Integration](#npc-integration)
8. [Session Management](#session-management)
9. [UI Rendering](#ui-rendering)
10. [YAML Configuration](#yaml-configuration)
11. [Best Practices](#best-practices)
12. [Examples](#examples)

---

## Core Concepts

### Dialogue Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      DialogueRegistry                           │
│  (Stores all DialogueTree definitions)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DialogueTree                              │
│  id: "blacksmith_main"                                          │
│  npcId: "blacksmith_001"                                        │
│  startNodeId: "greeting"                                        │
│  nodes: Map<String, DialogueNode>                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DialogueNode                               │
│  id: "greeting"                                                 │
│  speaker: "Thorin"                                              │
│  text: "Welcome to my forge!"                                   │
│  options: List<DialogueOption>                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     DialogueOption                              │
│  id: "shop"                                                     │
│  text: "Show me your wares"                                     │
│  targetNodeId: "shop_intro"                                     │
│  condition: Optional<DialogueCondition>                         │
│  action: Optional<DialogueAction>                               │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Description |
|-----------|-------------|
| `DialogueTree` | Complete conversation structure for an NPC |
| `DialogueNode` | Single point in conversation with speaker text |
| `DialogueOption` | Player response choice with navigation target |
| `DialogueCondition` | Predicate for option visibility/availability |
| `DialogueAction` | Code executed when option is selected |
| `DialogueSession` | Active conversation state for a player |
| `DialogueContext` | Runtime context with player/NPC data |

---

## Creating Dialogue Trees

### Using the Builder API

```java
import com.argonathsystems.framework.npc.dialogue.*;

DialogueTree dialogue = DialogueTree.builder("merchant_main")
    .npcId("merchant_001")
    .displayName("Marcus the Trader")
    .startNode("greeting")
    
    // Greeting node
    .node(DialogueNode.builder("greeting")
        .speaker("Marcus")
        .text("Welcome, traveler! Looking to buy or sell?")
        .option(DialogueOption.simple("buy", "What do you have for sale?", "shop_menu"))
        .option(DialogueOption.simple("sell", "I have items to sell", "sell_menu"))
        .option(DialogueOption.builder("quest")
            .text("Have you heard any rumors?")
            .targetNode("rumors")
            .condition(ctx -> ctx.hasReputation("merchants_guild", 50))
            .build())
        .option(DialogueOption.goodbye("Farewell"))
        .build())
    
    // Shop menu node
    .node(DialogueNode.builder("shop_menu")
        .speaker("Marcus")
        .text("Take a look at my finest goods!")
        .onEnter(ctx -> ctx.openShop("marcus_inventory"))
        .option(DialogueOption.simple("back", "Actually, let me think about it", "greeting"))
        .build())
    
    // Rumors node (reputation gated)
    .node(DialogueNode.builder("rumors")
        .speaker("Marcus")
        .text("*lowers voice* I've heard bandits are gathering in the eastern woods...")
        .onEnter(ctx -> ctx.unlockQuest("bandit_investigation"))
        .option(DialogueOption.simple("investigate", "Tell me more", "rumors_detail"))
        .option(DialogueOption.simple("back", "Interesting. Anything else?", "greeting"))
        .build())
    
    .build();
```

### Registering Dialogues

```java
// Get the dialogue registry
DialogueRegistry registry = npcFramework.getDialogueRegistry();

// Register the dialogue tree
registry.register(dialogue);

// Link to NPC
NPCDefinition merchant = NPCDefinitionBuilder.create("merchant_001")
    .displayName("Marcus the Trader")
    .dialogTreePath("merchant_main")  // References the dialogue tree ID
    .build();
```

---

## Dialogue Nodes

### Node Structure

```java
DialogueNode node = DialogueNode.builder("quest_complete")
    // Required fields
    .speaker("Elder Theron")
    .text("Excellent work! You have proven yourself worthy.")
    
    // Options for player responses
    .option(DialogueOption.simple("accept_reward", "Thank you, elder", "reward_given"))
    .option(DialogueOption.simple("decline", "The honor is enough", "decline_reward"))
    
    // Entry condition (must pass to enter this node)
    .entryCondition(ctx -> ctx.hasCompletedQuest("elder_quest_1"))
    
    // Actions triggered on node entry/exit
    .onEnter(ctx -> {
        ctx.giveReward("gold", 100);
        ctx.addReputation("village", 25);
    })
    .onExit(ctx -> {
        ctx.updateQuestState("elder_quest_1", "REWARDED");
    })
    
    // Auto-advance (for cutscenes/narration)
    .autoAdvance("next_node", 3.0f)  // Advance after 3 seconds
    
    .build();
```

### Node Types

#### Standard Node
Player sees text and chooses from options:
```java
DialogueNode.builder("standard")
    .speaker("NPC Name")
    .text("What would you like to do?")
    .option(...)
    .option(...)
    .build()
```

#### Narration Node
No speaker, often used for scene descriptions:
```java
DialogueNode.builder("narration")
    .text("*The room falls silent as you enter*")
    .autoAdvance("next_node", 2.0f)
    .build()
```

#### Action Node
Executes code and auto-advances:
```java
DialogueNode.builder("give_item")
    .speaker("Blacksmith")
    .text("Here is your sword!")
    .onEnter(ctx -> ctx.giveItem("iron_sword", 1))
    .autoAdvance("farewell", 1.5f)
    .build()
```

---

## Dialogue Options

### Option Types

#### Simple Option
Basic navigation to another node:
```java
DialogueOption.simple("option_id", "Display Text", "target_node_id")
```

#### Goodbye Option
Ends the dialogue:
```java
DialogueOption.goodbye("Farewell, friend")
```

#### Conditional Option
Only visible/available when condition passes:
```java
DialogueOption.builder("secret")
    .text("[Perception] I noticed something strange...")
    .targetNode("secret_info")
    .condition(ctx -> ctx.getPlayerStat("perception") >= 15)
    .build()
```

#### Action Option
Executes code when selected:
```java
DialogueOption.builder("give_gold")
    .text("[Give 50 gold] Here, take this for your troubles")
    .targetNode("grateful_response")
    .action(ctx -> {
        ctx.removeItem("gold", 50);
        ctx.addReputation("beggar", 100);
    })
    .build()
```

### Option Visibility

Options can be:
- **Visible & Available**: Condition passes → player can select
- **Visible & Unavailable**: Condition fails, `showWhenUnavailable=true` → grayed out
- **Hidden**: Condition fails, `showWhenUnavailable=false` → not shown

```java
DialogueOption.builder("expensive_option")
    .text("[Requires 1000 gold] I'll take your finest sword")
    .targetNode("expensive_purchase")
    .condition(ctx -> ctx.hasItem("gold", 1000))
    .showWhenUnavailable(true)  // Show grayed out if player lacks gold
    .unavailableText("(Not enough gold)")
    .build()
```

### Option Priority

Control option ordering with priority:
```java
DialogueOption.builder("important")
    .text("Critical response")
    .targetNode("important_path")
    .priority(100)  // Higher = shown first
    .build()
```

---

## Conditions and Actions

### DialogueCondition

A predicate that receives `DialogueContext` and returns boolean:

```java
// Inline condition
.condition(ctx -> ctx.getPlayerLevel() >= 10)

// Named condition for reuse
DialogueCondition hasCompletedIntro = ctx -> 
    ctx.hasCompletedQuest("intro_quest");

DialogueOption.builder("advanced")
    .text("I'm ready for a real challenge")
    .condition(hasCompletedIntro)
    .build()
```

### Common Condition Patterns

```java
// Quest state checks
ctx -> ctx.hasQuest("quest_id")
ctx -> ctx.hasCompletedQuest("quest_id")
ctx -> ctx.isQuestActive("quest_id")

// Reputation checks
ctx -> ctx.hasReputation("faction", minValue)
ctx -> ctx.getReputation("faction") >= 100

// Item checks
ctx -> ctx.hasItem("item_id")
ctx -> ctx.hasItem("gold", 500)

// Stat checks
ctx -> ctx.getPlayerStat("strength") >= 15
ctx -> ctx.getPlayerLevel() >= 20

// Custom variable checks
ctx -> ctx.hasVariable("met_before")
ctx -> ctx.getVariable("visits", 0) >= 3
```

### DialogueAction

Code executed when an option is selected or a node is entered/exited:

```java
// Give items
ctx -> ctx.giveItem("health_potion", 5)

// Remove items
ctx -> ctx.removeItem("gold", 100)

// Modify reputation
ctx -> ctx.addReputation("guild", 50)

// Start/complete quests
ctx -> ctx.startQuest("new_quest")
ctx -> ctx.completeQuest("current_quest")

// Set variables
ctx -> ctx.setVariable("chose_good_path", true)

// Open shop
ctx -> ctx.openShop("merchant_inventory")

// Teleport player
ctx -> ctx.teleport("tavern_entrance")

// Composite actions
ctx -> {
    ctx.giveItem("sword", 1);
    ctx.addReputation("blacksmith", 25);
    ctx.setVariable("received_sword", true);
}
```

---

## Quest Integration

### Quest-Aware Dialogues

```java
DialogueTree questGiverDialogue = DialogueTree.builder("elder_quests")
    .npcId("elder_001")
    .startNode("greeting")
    
    // Greeting changes based on quest state
    .node(DialogueNode.builder("greeting")
        .speaker("Elder Theron")
        .text(ctx -> {
            if (ctx.canCompleteQuest("elder_quest_1")) {
                return "You've returned! Tell me of your adventures.";
            } else if (ctx.isQuestActive("elder_quest_1")) {
                return "How fares your quest, young one?";
            } else if (ctx.hasCompletedQuest("elder_quest_1")) {
                return "Ah, my trusted friend. What brings you here?";
            } else {
                return "Welcome, stranger. Are you new to our village?";
            }
        })
        
        // Quest offer option (only if no active quest)
        .option(DialogueOption.builder("get_quest")
            .text("Do you have any work for me?")
            .targetNode("quest_offer")
            .condition(ctx -> !ctx.hasQuest("elder_quest_1") && 
                             !ctx.hasCompletedQuest("elder_quest_1"))
            .build())
        
        // Progress check option (only if quest active)
        .option(DialogueOption.builder("check_progress")
            .text("About the task you gave me...")
            .targetNode("quest_progress")
            .condition(ctx -> ctx.isQuestActive("elder_quest_1"))
            .build())
        
        // Turn in option (only if quest can complete)
        .option(DialogueOption.builder("complete_quest")
            .text("I've completed your task!")
            .targetNode("quest_complete")
            .condition(ctx -> ctx.canCompleteQuest("elder_quest_1"))
            .build())
        
        .option(DialogueOption.goodbye("Farewell"))
        .build())
    
    // Quest offer node
    .node(DialogueNode.builder("quest_offer")
        .speaker("Elder Theron")
        .text("The wolves have grown bold. Could you thin their numbers? " +
              "Bring me 10 wolf pelts as proof of your deed.")
        .onEnter(ctx -> ctx.showQuestPreview("elder_quest_1"))
        .option(DialogueOption.builder("accept")
            .text("I'll do it")
            .targetNode("quest_accepted")
            .action(ctx -> ctx.acceptQuest("elder_quest_1"))
            .build())
        .option(DialogueOption.simple("decline", "Not right now", "greeting"))
        .build())
    
    // Quest complete node
    .node(DialogueNode.builder("quest_complete")
        .speaker("Elder Theron")
        .text("Excellent! The village is in your debt. " +
              "Please accept this reward.")
        .onEnter(ctx -> {
            ctx.completeQuest("elder_quest_1");
            ctx.giveReward("gold", 200);
            ctx.giveItem("health_potion", 5);
            ctx.addReputation("village", 100);
        })
        .option(DialogueOption.simple("thanks", "Thank you, elder", "greeting"))
        .build())
    
    .build();
```

### Quest Accept/Complete in Dialogue

```java
// Accept quest through dialogue
DialogueOption.builder("accept_quest")
    .text("I accept your challenge")
    .targetNode("quest_details")
    .action(ctx -> ctx.acceptQuest("hero_quest_01"))
    .build()

// Complete quest through dialogue
.onEnter(ctx -> {
    if (ctx.canCompleteQuest("hero_quest_01")) {
        ctx.completeQuest("hero_quest_01");
        ctx.showRewardNotification();
    }
})
```

---

## NPC Integration

### Linking Dialogue to NPCs

```java
// Method 1: Through NPCDefinition
NPCDefinition npc = NPCDefinitionBuilder.create("blacksmith_01")
    .displayName("Thorin Ironforge")
    .dialogTreePath("blacksmith_main")  // Dialogue tree ID
    .build();

// Method 2: Dynamic dialogue selection
NPCDefinition npc = NPCDefinitionBuilder.create("guard_01")
    .displayName("City Guard")
    .dialogSelector(ctx -> {
        if (ctx.isWanted()) {
            return "guard_hostile";
        } else if (ctx.hasCompletedQuest("guard_favor")) {
            return "guard_friendly";
        } else {
            return "guard_neutral";
        }
    })
    .build();
```

### Handling NPC Interaction

```java
// In your event handler
eventAccessor.register(NPCInteractEvent.class, event -> {
    String npcId = event.getNpcId();
    UUID playerId = event.getPlayerId();
    
    // Get NPC definition
    NPCDefinition npc = npcManager.getDefinition(npcId);
    
    // Get appropriate dialogue tree
    String dialogueId = npc.getDialogTreePath()
        .orElseGet(() -> npc.getDialogSelector()
            .map(selector -> selector.select(createContext(playerId, npcId)))
            .orElse(null));
    
    if (dialogueId != null) {
        DialogueTree tree = dialogueRegistry.get(dialogueId);
        dialogueService.startDialogue(playerId, npcId, tree);
    }
});
```

---

## Session Management

### DialogueSession

Tracks active conversation state:

```java
public class DialogueSession {
    private final UUID playerId;
    private final String npcId;
    private final DialogueTree tree;
    private DialogueNode currentNode;
    private final DialogueContext context;
    private final Map<String, Object> sessionVariables;
    
    // Navigate to a node
    public void navigateTo(String nodeId) {
        DialogueNode node = tree.getNode(nodeId).orElseThrow();
        
        // Exit current node
        if (currentNode != null && currentNode.getOnExit() != null) {
            currentNode.getOnExit().execute(context);
        }
        
        currentNode = node;
        
        // Enter new node
        if (node.getOnEnter() != null) {
            node.getOnEnter().execute(context);
        }
    }
    
    // Get available options for current node
    public List<DialogueOption> getAvailableOptions() {
        return currentNode.getAvailableOptions(context);
    }
    
    // Select an option
    public void selectOption(String optionId) {
        DialogueOption option = currentNode.getOption(optionId);
        
        // Execute action
        option.getAction().ifPresent(action -> action.execute(context));
        
        // Navigate
        if (option.getTargetNodeId().isPresent()) {
            navigateTo(option.getTargetNodeId().get());
        } else {
            end();  // Goodbye option
        }
    }
}
```

### Session Service

```java
public class DialogueService {
    private final Map<UUID, DialogueSession> activeSessions = new ConcurrentHashMap<>();
    
    public DialogueSession startDialogue(UUID playerId, String npcId, DialogueTree tree) {
        // End any existing session
        endDialogue(playerId);
        
        // Create new session
        DialogueContext context = createContext(playerId, npcId);
        DialogueSession session = new DialogueSession(playerId, npcId, tree, context);
        
        // Start at the beginning
        session.navigateTo(tree.getStartNodeId());
        
        activeSessions.put(playerId, session);
        
        // Show UI
        dialogueUI.show(session);
        
        return session;
    }
    
    public void endDialogue(UUID playerId) {
        DialogueSession session = activeSessions.remove(playerId);
        if (session != null) {
            dialogueUI.hide(playerId);
        }
    }
}
```

---

## UI Rendering

### Dialogue UI Integration

The UI Framework handles rendering:

```java
public class DialogueUI {
    
    public void show(DialogueSession session) {
        DialogueNode node = session.getCurrentNode();
        List<DialogueOption> options = session.getAvailableOptions();
        
        // Create UI document (HyUIML)
        UIDocument doc = UIDocument.builder()
            .root(UIPanel.builder("dialogue_panel")
                .style("dialogue-container")
                
                // Speaker name
                .child(UILabel.builder("speaker")
                    .text(node.getSpeaker())
                    .style("dialogue-speaker")
                    .build())
                
                // Dialogue text
                .child(UILabel.builder("text")
                    .text(node.getText())
                    .style("dialogue-text")
                    .build())
                
                // Options panel
                .child(createOptionsPanel(session, options))
                
                .build())
            .build();
        
        uiService.show(session.getPlayerId(), doc);
    }
    
    private UIElement createOptionsPanel(DialogueSession session, List<DialogueOption> options) {
        UIPanel.Builder panel = UIPanel.builder("options")
            .style("dialogue-options");
        
        for (int i = 0; i < options.size(); i++) {
            DialogueOption option = options.get(i);
            
            UIButton button = UIButton.builder("option_" + i)
                .text(option.getText())
                .style(option.isAvailable(session.getContext()) ? 
                       "dialogue-option" : "dialogue-option-disabled")
                .onClick(() -> session.selectOption(option.getId()))
                .enabled(option.isAvailable(session.getContext()))
                .build();
            
            panel.child(button);
        }
        
        return panel.build();
    }
}
```

---

## YAML Configuration

### Dialogue Definition in YAML

```yaml
# dialogues/blacksmith_main.yml
id: blacksmith_main
npc_id: blacksmith_001
display_name: "Blacksmith Conversation"
start_node: greeting

nodes:
  greeting:
    speaker: "Thorin"
    text: "Welcome to my forge! What can I do for you?"
    options:
      - id: shop
        text: "Show me your wares"
        target_node: shop_intro
      - id: repair
        text: "Can you repair my equipment?"
        target_node: repair_menu
        condition:
          type: has_item
          item: damaged_weapon
      - id: quest
        text: "I heard you need help with something"
        target_node: quest_offer
        condition:
          type: quest_not_started
          quest_id: blacksmith_quest_1
      - id: goodbye
        text: "Farewell"
        ends_dialogue: true
  
  shop_intro:
    speaker: "Thorin"
    text: "Take a look at my finest work!"
    on_enter:
      - action: open_shop
        shop_id: thorin_inventory
    options:
      - id: back
        text: "Let me think about it"
        target_node: greeting
  
  quest_offer:
    speaker: "Thorin"
    text: "I need rare ore from the abandoned mine. Bring me 10 Mythril Ore."
    on_enter:
      - action: show_quest_preview
        quest_id: blacksmith_quest_1
    options:
      - id: accept
        text: "I'll bring you the ore"
        target_node: quest_accepted
        action:
          - type: accept_quest
            quest_id: blacksmith_quest_1
      - id: decline
        text: "Sounds dangerous..."
        target_node: greeting

  quest_accepted:
    speaker: "Thorin"
    text: "Excellent! Be careful in those mines."
    options:
      - id: goodbye
        text: "I'll return soon"
        ends_dialogue: true
```

### Loading YAML Dialogues

```java
// Load all dialogues from directory
Path dialoguesDir = Paths.get("config/dialogues");
DialogueYamlLoader loader = new DialogueYamlLoader(configManager);

Files.walk(dialoguesDir)
    .filter(p -> p.toString().endsWith(".yml"))
    .forEach(path -> {
        DialogueTree tree = loader.load(path);
        dialogueRegistry.register(tree);
    });
```

---

## Best Practices

### DO:

- ✅ **Use meaningful node IDs** - `quest_complete` not `node_47`
- ✅ **Keep dialogue text concise** - Players skim long text
- ✅ **Test all branches** - Every option should lead somewhere valid
- ✅ **Validate dialogue trees** - Use `DialogueTree.validate()`
- ✅ **Use conditions sparingly** - Too many hidden options confuse players
- ✅ **Provide context** - Mark skill/item checks in option text `[Perception]`

### DON'T:

- ❌ **Create orphan nodes** - Nodes with no path leading to them
- ❌ **Forget goodbye options** - Always provide an exit
- ❌ **Make deep nesting** - Keep conversation trees relatively flat
- ❌ **Overuse auto-advance** - Let players read at their pace
- ❌ **Hard-code strings** - Use localization for text

### Validation

```java
// Validate dialogue structure
Map<String, String> issues = dialogue.validate();
if (!issues.isEmpty()) {
    for (Map.Entry<String, String> issue : issues.entrySet()) {
        logger.error("Dialogue issue at {}: {}", issue.getKey(), issue.getValue());
    }
    throw new DialogueValidationException("Dialogue has structural issues");
}
```

---

## Examples

### Example 1: Simple Merchant

```java
DialogueTree merchantDialogue = DialogueTree.builder("simple_merchant")
    .npcId("merchant_001")
    .startNode("greeting")
    .node(DialogueNode.builder("greeting")
        .speaker("Merchant")
        .text("Welcome! Would you like to see my goods?")
        .option(DialogueOption.builder("shop")
            .text("Show me what you have")
            .action(ctx -> ctx.openShop("merchant_shop"))
            .build())
        .option(DialogueOption.goodbye("No thanks"))
        .build())
    .build();
```

### Example 2: Branching Conversation

```java
DialogueTree branchingDialogue = DialogueTree.builder("moral_choice")
    .npcId("prisoner")
    .startNode("plea")
    .node(DialogueNode.builder("plea")
        .speaker("Prisoner")
        .text("Please, help me escape! I was falsely imprisoned!")
        .option(DialogueOption.builder("help")
            .text("[Good] I'll help you")
            .targetNode("gratitude")
            .action(ctx -> ctx.setVariable("helped_prisoner", true))
            .build())
        .option(DialogueOption.builder("refuse")
            .text("[Neutral] I can't risk it")
            .targetNode("understanding")
            .build())
        .option(DialogueOption.builder("betray")
            .text("[Evil] Guards! Escape attempt!")
            .targetNode("betrayal")
            .action(ctx -> ctx.addReputation("guards", 50))
            .build())
        .build())
    // ... additional nodes for each path
    .build();
```

### Example 3: Skill Check Dialogue

```java
.option(DialogueOption.builder("persuade")
    .text("[Persuasion 15] I'm sure we can work something out...")
    .targetNode("persuaded")
    .condition(ctx -> ctx.getPlayerStat("persuasion") >= 15)
    .showWhenUnavailable(true)
    .unavailableText("[Requires Persuasion 15]")
    .action(ctx -> ctx.grantXP("persuasion", 50))
    .build())
```

---

## Related Documentation

- [NPC Framework API Reference](../api-reference/npc-framework.md)
- [NPC Integration Guide](./npc-integration.md)
- [Quest Development Guide](./quest-development.md)
- [UI Development Guide](./ui-development.md)
- [Hytale Adapter Integration](./hytale-adapter-integration.md)
