# Quick Start Guide

Get up and running with Argonath Systems in 5 minutes!

## Prerequisites

- Java 17 or higher
- Maven 3.8+
- Your favorite IDE (IntelliJ IDEA recommended)
- Basic understanding of Java and Maven

## Option 1: Using Bundles (Recommended for Beginners)

### Step 1: Add Bundle Dependency

Add to your `pom.xml`:

```xml
<dependencies>
    <!-- For quest-based mods -->
    <dependency>
        <groupId>com.hytalemod.argonath</groupId>
        <artifactId>bundle-quest</artifactId>
        <version>1.0.0</version>
    </dependency>
</dependencies>
```

### Step 2: Initialize the Framework

```java
import com.hytalemod.argonath.bundle.quest.QuestBundle;

public class MyMod {
    private QuestBundle questBundle;
    
    public void onModInit() {
        questBundle = QuestBundle.builder()
            .withDefaults()
            .build();
            
        questBundle.initialize();
    }
}
```

### Step 3: Create Your First Quest

```java
import com.hytalemod.argonath.framework.quest.Quest;
import com.hytalemod.argonath.framework.objective.ObjectiveType;

Quest myFirstQuest = questBundle.getQuestManager()
    .createQuest("first_quest")
    .withName("Welcome to Argonath!")
    .withDescription("Complete your first quest")
    .addObjective(ObjectiveType.TALK_TO_NPC, "village_elder")
    .addReward("gold", 100)
    .build();

// Register the quest
questBundle.getQuestRegistry().register(myFirstQuest);
```

### Step 4: Test It!

That's it! You now have a working quest system.

## Option 2: Individual Modules (For Advanced Users)

### Step 1: Choose Your Modules

Minimal quest setup requires:
- `platform-core`
- `framework-core`
- `framework-quest`
- `framework-objective`

### Step 2: Add Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>com.hytalemod.argonath</groupId>
        <artifactId>platform-core</artifactId>
        <version>1.0.0</version>
    </dependency>
    <dependency>
        <groupId>com.hytalemod.argonath</groupId>
        <artifactId>framework-quest</artifactId>
        <version>1.0.0</version>
    </dependency>
    <!-- Add other modules as needed -->
</dependencies>
```

### Step 3: Configure Manually

```java
import com.hytalemod.argonath.platform.core.ServiceRegistry;
import com.hytalemod.argonath.framework.quest.QuestManager;

ServiceRegistry registry = ServiceRegistry.create();
QuestManager questManager = new QuestManager(registry);
questManager.initialize();
```

## What's Next?

### Learn the Basics
- **[Installation Guide](installation.md)** - Detailed setup and configuration
- **[Your First Quest](first-quest.md)** - Build a complete quest chain
- **[Bundles vs Modules](bundles-vs-modules.md)** - Choose the right approach

### Explore Features
- **[Quest Development Guide](../guides/quest-development.md)** - Advanced quest patterns
- **[NPC Integration](../guides/npc-integration.md)** - Add interactive NPCs
- **[UI Development](../guides/ui-development.md)** - Custom quest trackers

### Join the Community
- Browse [example projects](../../examples/)
- Ask questions in GitHub Discussions
- Share your creations!

## Common Issues

### Build Fails
Ensure you have Java 17+ and Maven 3.8+:
```bash
java -version
mvn -version
```

### Module Not Found
Check that your repository includes the Argonath repository:
```xml
<repositories>
    <repository>
        <id>argonath-systems</id>
        <url>https://maven.pkg.github.com/yourusername/Argonath-Systems</url>
    </repository>
</repositories>
```

### Runtime Errors
Enable debug logging:
```java
questBundle.builder()
    .withLogLevel(LogLevel.DEBUG)
    .build();
```

## Examples

Check out complete examples in the [examples directory](../../examples/):
- `simple-quest-mod` - Basic quest implementation
- `npc-dialogue-mod` - Interactive NPCs
- `quest-chain-mod` - Multi-quest storyline

---

**Need Help?** See the [FAQ](../faq.md) or ask in [Discussions](https://github.com/yourusername/Argonath-Systems/discussions).
