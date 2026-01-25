# Module Integration Guide

Learn how to combine multiple Argonath frameworks to create rich, interconnected features.

## Integration Patterns

### Pattern 1: Quest + NPC Integration

Create quests that interact with NPCs for dialogues and quest givers.

#### Dependencies
```xml
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-quest</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-npc</artifactId>
    <version>1.0.0</version>
</dependency>
```

#### Implementation
```java
// Create an NPC quest giver
NPC questGiver = npcManager.createNPC("village_elder")
    .withName("Village Elder")
    .withDialogue("greeting", "Welcome, traveler!")
    .build();

// Create a quest from this NPC
Quest quest = questManager.createQuest("elder_quest")
    .withQuestGiver(questGiver.getId())
    .addObjective(ObjectiveType.TALK_TO_NPC, questGiver.getId())
    .onComplete(() -> {
        questGiver.updateDialogue("greeting", "Thank you for your help!");
    })
    .build();

// Link NPC interaction to quest
npcManager.onInteract(questGiver.getId(), (player, npc) -> {
    if (!questManager.hasQuest(player, quest.getId())) {
        questManager.offerQuest(player, quest);
    } else {
        questManager.advanceObjective(player, quest.getId());
    }
});
```

### Pattern 2: Quest + UI Integration

Display quest information in custom UI elements.

#### Dependencies
```xml
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-quest</artifactId>
</dependency>
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-ui</artifactId>
</dependency>
```

#### Implementation
```java
// Create a quest tracker UI component
UIComponent questTracker = uiManager.createPanel("quest_tracker")
    .withTitle("Active Quests")
    .withSize(300, 400)
    .build();

// Bind quest data to UI
questManager.onQuestAccepted((player, quest) -> {
    questTracker.addChild(
        uiManager.createLabel(quest.getId())
            .withText(quest.getName())
            .withDescription(quest.getDescription())
            .build()
    );
});

questManager.onQuestCompleted((player, quest) -> {
    questTracker.removeChild(quest.getId());
});

// Update UI on objective progress
questManager.onObjectiveProgress((player, quest, objective) -> {
    questTracker.updateChild(quest.getId(), 
        label -> label.setProgress(objective.getProgress())
    );
});
```

### Pattern 3: Condition + Quest Integration

Use conditions to control quest availability and progression.

#### Dependencies
```xml
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-quest</artifactId>
</dependency>
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-condition</artifactId>
</dependency>
```

#### Implementation
```java
// Define quest prerequisites using conditions
Condition levelRequirement = conditionManager.create()
    .type(ConditionType.PLAYER_LEVEL)
    .withParameter("minLevel", 10)
    .build();

Condition previousQuestCompleted = conditionManager.create()
    .type(ConditionType.QUEST_COMPLETED)
    .withParameter("questId", "intro_quest")
    .build();

// Create quest with conditions
Quest advancedQuest = questManager.createQuest("advanced_quest")
    .withName("Advanced Challenge")
    .withPrerequisites(levelRequirement, previousQuestCompleted)
    .build();

// Use conditions for branching quest paths
Quest moralChoice = questManager.createQuest("moral_choice")
    .addObjective(ObjectiveType.CHOICE, "help_or_harm")
    .onChoice("help", () -> {
        questManager.createFollowUpQuest("hero_path")
            .withPrerequisite(
                conditionManager.questChoice("moral_choice", "help")
            )
            .build();
    })
    .onChoice("harm", () -> {
        questManager.createFollowUpQuest("villain_path")
            .withPrerequisite(
                conditionManager.questChoice("moral_choice", "harm")
            )
            .build();
    })
    .build();
```

### Pattern 4: Storage + Framework Integration

Persist quest and NPC data across sessions.

#### Dependencies
```xml
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-quest</artifactId>
</dependency>
<dependency>
    <groupId>com.hytalemod.argonath</groupId>
    <artifactId>framework-storage</artifactId>
</dependency>
```

#### Implementation
```java
// Configure quest storage
storageManager.register(QuestProgress.class)
    .withSerializer(new QuestProgressSerializer())
    .withBackend(StorageBackend.JSON)
    .build();

// Auto-save quest progress
questManager.onQuestProgress((player, quest) -> {
    QuestProgress progress = questManager.getProgress(player, quest);
    storageManager.save("quest_progress", player.getId(), progress);
});

// Load quest progress on player join
playerManager.onPlayerJoin(player -> {
    QuestProgress progress = storageManager.load(
        "quest_progress", 
        player.getId(), 
        QuestProgress.class
    );
    
    if (progress != null) {
        questManager.restoreProgress(player, progress);
    }
});
```

### Pattern 5: Config + Multi-Framework Setup

Centralize configuration for multiple frameworks.

#### Configuration File (config/argonath.yml)
```yaml
quest:
  maxActiveQuests: 5
  autoSave: true
  saveInterval: 300 # seconds

npc:
  maxInteractionDistance: 5.0
  enableDialogueHistory: true

ui:
  questTracker:
    enabled: true
    position:
      x: 10
      y: 10
    size:
      width: 300
      height: 400

storage:
  backend: JSON
  path: ./data/quests
```

#### Implementation
```java
// Load central configuration
Config config = configManager.load("argonath.yml");

// Configure quest manager
questManager.configure(builder -> builder
    .maxActiveQuests(config.getInt("quest.maxActiveQuests"))
    .autoSave(config.getBoolean("quest.autoSave"))
    .saveInterval(config.getInt("quest.saveInterval"))
);

// Configure NPC manager
npcManager.configure(builder -> builder
    .maxInteractionDistance(config.getDouble("npc.maxInteractionDistance"))
    .enableDialogueHistory(config.getBoolean("npc.enableDialogueHistory"))
);

// Configure UI
if (config.getBoolean("ui.questTracker.enabled")) {
    uiManager.createQuestTracker()
        .atPosition(
            config.getInt("ui.questTracker.position.x"),
            config.getInt("ui.questTracker.position.y")
        )
        .withSize(
            config.getInt("ui.questTracker.size.width"),
            config.getInt("ui.questTracker.size.height")
        )
        .build();
}
```

## Complete Integration Example

### Scenario: Full Quest System with NPCs, UI, and Persistence

```java
public class CompleteQuestMod {
    private ServiceRegistry registry;
    private QuestManager questManager;
    private NPCManager npcManager;
    private UIManager uiManager;
    private ConditionManager conditionManager;
    private StorageManager storageManager;
    
    public void initialize() {
        // 1. Initialize service registry
        registry = ServiceRegistry.create();
        
        // 2. Initialize managers
        storageManager = new StorageManager(registry);
        conditionManager = new ConditionManager(registry);
        npcManager = new NPCManager(registry);
        questManager = new QuestManager(registry);
        uiManager = new UIManager(registry);
        
        // 3. Configure storage
        configureStorage();
        
        // 4. Create NPCs
        NPC questGiver = createQuestGiver();
        
        // 5. Create quests
        Quest mainQuest = createMainQuest(questGiver);
        
        // 6. Set up UI
        createQuestTrackerUI();
        
        // 7. Wire everything together
        wireEventHandlers();
    }
    
    private void configureStorage() {
        storageManager.register(QuestProgress.class)
            .withBackend(StorageBackend.JSON)
            .build();
    }
    
    private NPC createQuestGiver() {
        return npcManager.createNPC("elder")
            .withName("Village Elder")
            .withDialogue("greeting", "Greetings, adventurer!")
            .withDialogue("quest_available", "I need your help!")
            .withDialogue("quest_complete", "Thank you!")
            .build();
    }
    
    private Quest createMainQuest(NPC questGiver) {
        Condition levelReq = conditionManager.create()
            .type(ConditionType.PLAYER_LEVEL)
            .withParameter("minLevel", 5)
            .build();
            
        return questManager.createQuest("main_quest")
            .withName("The Elder's Request")
            .withDescription("Help the village elder")
            .withQuestGiver(questGiver.getId())
            .withPrerequisite(levelReq)
            .addObjective(ObjectiveType.TALK_TO_NPC, questGiver.getId())
            .addObjective(ObjectiveType.COLLECT_ITEM, "ancient_scroll", 1)
            .addObjective(ObjectiveType.RETURN_TO_NPC, questGiver.getId())
            .addReward("gold", 500)
            .addReward("experience", 1000)
            .build();
    }
    
    private void createQuestTrackerUI() {
        UIComponent tracker = uiManager.createPanel("quest_tracker")
            .withTitle("Active Quests")
            .atPosition(10, 10)
            .withSize(300, 400)
            .build();
            
        questManager.onQuestAccepted((player, quest) -> {
            tracker.addQuestDisplay(quest);
        });
        
        questManager.onObjectiveProgress((player, quest, objective) -> {
            tracker.updateObjectiveProgress(quest.getId(), objective);
        });
        
        questManager.onQuestCompleted((player, quest) -> {
            tracker.removeQuestDisplay(quest.getId());
        });
    }
    
    private void wireEventHandlers() {
        // Save progress on changes
        questManager.onQuestProgress((player, quest) -> {
            storageManager.save("progress", player.getId(), 
                questManager.getProgress(player, quest));
        });
        
        // Load progress on join
        registry.getEventBus().on(PlayerJoinEvent.class, event -> {
            QuestProgress progress = storageManager.load(
                "progress", 
                event.getPlayer().getId(),
                QuestProgress.class
            );
            questManager.restoreProgress(event.getPlayer(), progress);
        });
    }
}
```

## Best Practices

### 1. Use Service Registry
Always register your managers with the service registry for proper dependency injection:

```java
registry.register(QuestManager.class, questManager);
registry.register(NPCManager.class, npcManager);
```

### 2. Event-Driven Integration
Prefer events over direct coupling:

```java
// Good: Event-driven
questManager.onQuestComplete((player, quest) -> {
    npcManager.updateDialogue(...);
});

// Avoid: Direct coupling
questManager.completeQuest(player, quest, npcManager);
```

### 3. Configuration Externalization
Keep configuration external for flexibility:

```java
// Good: Configurable
int maxQuests = config.getInt("quest.maxActiveQuests");

// Avoid: Hardcoded
int maxQuests = 5;
```

### 4. Lifecycle Management
Initialize in correct order (dependencies first):

```java
1. Platform Core
2. Storage & Config
3. Condition System
4. Individual Frameworks (NPC, Quest)
5. UI Components
6. Wiring/Event Handlers
```

## Troubleshooting

### Issue: Circular Dependencies
**Solution**: Use event bus to break circular dependencies.

### Issue: Service Not Found
**Solution**: Ensure proper initialization order and service registration.

### Issue: Data Not Persisting
**Solution**: Verify storage backend configuration and serializers.

## Next Steps

- **[Quest Development Guide](../guides/quest-development.md)** - Build complex quests
- **[NPC Integration](../guides/npc-integration.md)** - Advanced NPC features
- **[UI Development](../guides/ui-development.md)** - Custom UI components

---

**Ready to integrate?** Start with the [Complete Example](../../examples/complete-integration/) project!
