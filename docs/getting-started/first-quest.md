# Your First Quest

Complete tutorial for creating your first quest using Argonath Systems.

## What You'll Build

A simple "Welcome to the World" quest that:
- Appears when the player joins
- Requires talking to an NPC
- Gives a reward upon completion
- Demonstrates core quest concepts

**Estimated Time:** 15 minutes

## Prerequisites

- ✅ Argonath Systems installed ([Installation Guide](installation.md))
- ✅ Quest Bundle or individual quest modules
- ✅ Basic Java knowledge
- ✅ Development environment set up

## Project Setup

### Step 1: Create Your Mod Project

```bash
# Create mod structure
mkdir my-first-quest
cd my-first-quest
```

Create `build.gradle`:

```gradle
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = JavaVersion.VERSION_17

repositories {
    mavenCentral()
    maven {
        url = 'https://maven.pkg.github.com/YOUR_USERNAME/Argonath-Systems'
    }
}

dependencies {
    // Use the Quest Bundle
    implementation 'com.hytale.argonath:bundle-quest:1.0.0'
    
    // Or individual modules
    // implementation 'com.hytale.argonath:framework-quest:1.0.0'
}
```

### Step 2: Create Package Structure

```
src/main/java/com/example/firstquest/
├── FirstQuestMod.java
├── quests/
│   └── WelcomeQuest.java
└── npcs/
    └── VillageElderNPC.java
```

## Creating the Quest

### Step 3: Define the Quest Class

Create `WelcomeQuest.java`:

```java
package com.example.firstquest.quests;

import com.hytale.argonath.framework.quest.Quest;
import com.hytale.argonath.framework.quest.QuestBuilder;
import com.hytale.argonath.framework.objective.Objective;
import com.hytale.argonath.framework.objective.ObjectiveType;
import com.hytale.argonath.framework.condition.Condition;
import com.hytale.argonath.framework.npc.NPC;

public class WelcomeQuest {
    
    public static Quest create() {
        return QuestBuilder.create("welcome_quest")
            .name("Welcome to the World")
            .description("The Village Elder wants to speak with you.")
            
            // Quest appears when player joins for the first time
            .startCondition(Condition.firstJoin())
            
            // Objective: Talk to the Village Elder
            .addObjective(
                Objective.builder()
                    .type(ObjectiveType.TALK_TO_NPC)
                    .target("village_elder")
                    .description("Talk to the Village Elder")
                    .build()
            )
            
            // Reward
            .addReward(reward -> reward
                .item("gold_coin", 10)
                .experience(50)
            )
            
            // Quest dialog
            .onComplete(player -> {
                player.sendMessage("§a[Elder] Welcome, young adventurer! " +
                    "May your journey be filled with glory!");
            })
            
            .build();
    }
}
```

### Step 4: Create the NPC

Create `VillageElderNPC.java`:

```java
package com.example.firstquest.npcs;

import com.hytale.argonath.framework.npc.NPC;
import com.hytale.argonath.framework.npc.NPCBuilder;
import com.hytale.argonath.framework.npc.DialogNode;

public class VillageElderNPC {
    
    public static NPC create() {
        return NPCBuilder.create("village_elder")
            .name("Village Elder")
            .model("models/npc/elder.json")
            .location(100, 64, 200) // x, y, z coordinates
            
            // Dialog when quest is active
            .addDialog(
                DialogNode.builder()
                    .condition("quest_active:welcome_quest")
                    .text("Greetings, traveler! Welcome to our village.")
                    .option("Hello! What can I do?", "quest_accept")
                    .build()
            )
            
            // Dialog after quest completion
            .addDialog(
                DialogNode.builder()
                    .condition("quest_completed:welcome_quest")
                    .text("I see you've settled in nicely!")
                    .build()
            )
            
            // Quest completion handler
            .onInteract((npc, player) -> {
                // Automatically handled by Quest Framework
            })
            
            .build();
    }
}
```

### Step 5: Register in Main Mod Class

Create `FirstQuestMod.java`:

```java
package com.example.firstquest;

import com.hytale.argonath.platform.sdk.ModInitializer;
import com.hytale.argonath.framework.quest.QuestRegistry;
import com.hytale.argonath.framework.npc.NPCRegistry;
import com.example.firstquest.quests.WelcomeQuest;
import com.example.firstquest.npcs.VillageElderNPC;

public class FirstQuestMod implements ModInitializer {
    
    @Override
    public void onInitialize() {
        // Register the quest
        QuestRegistry.register(WelcomeQuest.create());
        
        // Register the NPC
        NPCRegistry.register(VillageElderNPC.create());
        
        System.out.println("[First Quest] Mod initialized!");
    }
}
```

### Step 6: Create Mod Metadata

Create `src/main/resources/mod.json`:

```json
{
  "id": "first_quest",
  "name": "My First Quest",
  "version": "1.0.0",
  "description": "A simple welcome quest",
  "authors": ["YourName"],
  "entrypoints": {
    "main": ["com.example.firstquest.FirstQuestMod"]
  },
  "depends": {
    "argonath-quest": ">=1.0.0"
  }
}
```

## Building and Testing

### Step 7: Build the Mod

```bash
./gradlew build
```

Output JAR: `build/libs/my-first-quest-1.0.0.jar`

### Step 8: Install and Test

```bash
# Copy to Hytale mods folder
cp build/libs/my-first-quest-1.0.0.jar ~/hytale/mods/

# Launch Hytale
```

### Step 9: Verify in Game

1. **Join the world** (as a new player)
2. **Check quest log**: `/quests` or press `Q`
3. **See "Welcome to the World"** quest active
4. **Find the Village Elder** at coordinates (100, 64, 200)
5. **Talk to the NPC** - quest should complete
6. **Check rewards** - 10 gold coins, 50 XP

## Understanding What You Built

### Quest Flow

```
Player Joins (First Time)
    ↓
Quest Activates Automatically
    ↓
Quest Appears in Quest Log
    ↓
Player Finds Village Elder NPC
    ↓
Player Talks to NPC
    ↓
Objective Completes
    ↓
Rewards Given
    ↓
Quest Marked Complete
```

### Key Components

#### 1. **QuestBuilder**
Fluent API for creating quests:
- `name()` - Display name
- `description()` - Quest description
- `startCondition()` - When quest becomes available
- `addObjective()` - What player must do
- `addReward()` - What player receives

#### 2. **Objectives**
Track player progress:
- `TALK_TO_NPC` - Interact with specific NPC
- `KILL_ENTITY` - Defeat enemies
- `COLLECT_ITEM` - Gather items
- `REACH_LOCATION` - Travel to area

#### 3. **Conditions**
Control quest availability:
- `firstJoin()` - New players only
- `hasQuest()` - Prerequisite quests
- `hasItem()` - Inventory requirements
- `inLocation()` - Area requirements

#### 4. **NPCs**
Interactive characters:
- `addDialog()` - Conversation trees
- `condition()` - Context-aware responses
- `onInteract()` - Custom behavior

## Enhancing Your Quest

### Add Multiple Objectives

```java
.addObjective(
    Objective.builder()
        .type(ObjectiveType.TALK_TO_NPC)
        .target("village_elder")
        .build()
)
.addObjective(
    Objective.builder()
        .type(ObjectiveType.COLLECT_ITEM)
        .target("apple")
        .count(5)
        .description("Collect 5 apples")
        .build()
)
```

### Add Quest Chain

```java
// Second quest unlocks after first
.startCondition(
    Condition.questCompleted("welcome_quest")
)
```

### Custom Rewards

```java
.addReward(reward -> reward
    .item("gold_coin", 10)
    .item("iron_sword", 1)
    .experience(100)
    .title("Village Friend")
    .custom(player -> {
        // Custom reward logic
        player.unlockAchievement("first_quest");
    })
)
```

### Dialog Choices

```java
.addDialog(
    DialogNode.builder()
        .text("Need help with something?")
        .option("Tell me about the village", "village_info")
        .option("Any quests for me?", "quest_offer")
        .option("Goodbye", "end")
        .build()
)
```

## Common Patterns

### Time-Limited Quest

```java
.startCondition(Condition.timeOfDay(6, 18)) // 6 AM to 6 PM
.expiresIn(Duration.ofHours(1))
```

### Level Requirement

```java
.startCondition(Condition.level(5))
```

### Group Quest

```java
.requiresParty(true)
.partySize(2, 4) // 2-4 players
```

## Debugging Tips

### Enable Debug Logging

In `config/argonath/platform.json`:
```json
{
  "debug": true,
  "logLevel": "DEBUG"
}
```

### Check Quest Status

```
/argonath quest status welcome_quest
/argonath quest complete welcome_quest  # Force complete for testing
/argonath quest reset welcome_quest     # Reset for retesting
```

### Common Issues

**Quest not appearing?**
- Check start conditions
- Verify quest is registered
- Check logs for errors

**NPC not spawning?**
- Verify coordinates
- Check NPC model path
- Ensure NPC is registered

**Objective not completing?**
- Verify target IDs match
- Check objective type
- Enable debug logging

## Next Steps

Now that you've created your first quest, explore:

- **[Quest Development Guide](../guides/quest-development.md)** - Advanced quest patterns
- **[NPC Integration](../guides/npc-integration.md)** - Complex NPC behavior
- **[Bundles vs Modules](bundles-vs-modules.md)** - Choose your approach
- **[API Reference](../api/framework-quest.md)** - Full quest API

## Complete Example Code

Full working example available at:
**[examples/first-quest](https://github.com/YOUR_USERNAME/Argonath-Systems/tree/main/examples/first-quest)**

---

**Congratulations!** You've created your first quest with Argonath Systems. 🎉
