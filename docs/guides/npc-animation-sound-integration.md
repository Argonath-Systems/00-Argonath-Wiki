# NPC Animation & Sound Integration Guide

> **Specification Reference**: SF-NPC-041: NPC Audio Integration  
> **Implementation Plan**: IMPL-PLAN-2026-Q1-NPC-QUEST-ANIMATION

This guide covers the animation and sound integration system for NPCs, enabling synchronized audio-visual feedback for NPC events.

## Overview

The NPC Animation & Sound Integration system provides:

- **Auto-Discovery**: Automatically detect animations from model assets on NPC spawn
- **28 Animation Triggers**: Lifecycle, dialogue, quest, trade, social, combat, movement events
- **Synchronized Playback**: Animations play with associated sounds
- **Cooldown Management**: Prevent animation spam with configurable cooldowns
- **Quest Integration**: Automatic animation triggers on quest events

## Key Components

### SynchronizedAnimationSound

A record that pairs an animation with a sound event for synchronized playback.

```java
// From a native model animation-sound pair
SynchronizedAnimationSound death = SynchronizedAnimationSound.fromNative(
    new AnimationSoundPair("anim:death", "sfx:death_groan"),
    AnimationTrigger.ON_DEATH
);

// Animation only (no sound)
SynchronizedAnimationSound wave = SynchronizedAnimationSound.animationOnly(
    "anim:wave",
    AnimationTrigger.ON_GREETING
);

// Sound only (no animation)
SynchronizedAnimationSound ambient = SynchronizedAnimationSound.soundOnly(
    "sfx:ambient_hum",
    AnimationTrigger.RANDOM
);

// Both with custom settings
SynchronizedAnimationSound custom = new SynchronizedAnimationSound(
    "anim:attack",
    "sfx:sword_swing",
    AnimationTrigger.ON_COMBAT_START,
    0.8f,   // volume
    0.2f,   // pitch variation (±20%)
    true,   // interruptible
    1000    // 1 second cooldown
);
```

### NPCAnimationBehavior Configuration

Configure animation behavior for an NPC:

```java
// Auto-discovery with defaults
NPCAnimationBehavior behavior = NPCAnimationBehavior.defaults();

// Manual configuration
NPCAnimationBehavior manual = NPCAnimationBehavior.builder()
    .disableAutoDiscovery()
    .onDeath("anim:death_fall", "sfx:death_scream")
    .onQuestComplete("anim:celebrate", "sfx:fanfare")
    .onDialogueStart("anim:attention", null)
    .onGreeting("anim:wave", "sfx:hello")
    .onTrigger(AnimationTrigger.RANDOM, "anim:fidget", null)
    .build();

// Hybrid - auto-discover with overrides
NPCAnimationBehavior hybrid = NPCAnimationBehavior.builder()
    .autoDiscoverAnimations(true)
    .animationSet("humanoid_villager")
    .onDeath("anim:custom_death", "sfx:custom_death") // Override discovered
    .build();
```

### NPCAnimationSoundService

The runtime service that manages animation execution:

```java
// Inject the service
@Inject
NPCAnimationSoundService animationService;

// On NPC spawn - register animations
animationService.onNPCSpawn(
    npc.getId(),
    npc.getModelAssetId(),
    behavior.isAutoDiscoverAnimations(),
    behavior.getTriggerMap()
);

// Trigger an animation
animationService.trigger(npc, AnimationTrigger.ON_GREETING);

// On NPC despawn - cleanup
animationService.onNPCDespawn(npc.getId());
```

## Animation Triggers

The system supports 28 different triggers organized by category:

### Lifecycle Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_SPAWN` | NPC spawns into the world |
| `ON_DESPAWN` | NPC despawns |
| `ON_DEATH` | NPC dies |

### Interaction Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_INTERACT` | Player interacts with NPC |
| `ON_PLAYER_APPROACH` | Player approaches NPC |
| `ON_FIRST_MEET` | First time player meets NPC |
| `ON_GREETING` | NPC greets player |

### Dialogue Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_DIALOGUE_START` | Dialogue begins |
| `ON_DIALOGUE_END` | Dialogue ends normally |
| `ON_DIALOGUE_ABANDON` | Player abandons dialogue |
| `ON_DIALOGUE_CHOICE` | Player makes a dialogue choice |
| `ON_DIALOGUE_LINE` | NPC speaks a line |

### Quest Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_QUEST_OFFER` | NPC offers a quest |
| `ON_QUEST_ACCEPT` | Player accepts quest |
| `ON_QUEST_COMPLETE` | Quest objectives completed |
| `ON_QUEST_TURN_IN` | Player turns in quest |
| `ON_QUEST_FAIL` | Quest fails |

### Trade Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_TRADE_START` | Trade begins |
| `ON_TRADE_COMPLETE` | Trade completes |
| `ON_TRADE_CANCEL` | Trade is cancelled |

### Combat Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_COMBAT_START` | Combat begins |
| `ON_COMBAT_END` | Combat ends |
| `ON_DAMAGE_TAKEN` | NPC takes damage |
| `ON_DAMAGE_DEALT` | NPC deals damage |
| `ON_BECOME_HOSTILE` | NPC becomes hostile |
| `ON_BECOME_FRIENDLY` | NPC becomes friendly |

### Movement Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_MOVEMENT_START` | NPC starts moving |
| `ON_MOVEMENT_STOP` | NPC stops moving |
| `ON_WAYPOINT_REACHED` | NPC reaches patrol waypoint |
| `ON_MOVEMENT_MODE_CHANGE` | Movement mode changes (walk/run) |

### Social Triggers
| Trigger | When Fired |
|---------|------------|
| `ON_AFFINITY_CHANGE` | Relationship with player changes |

### Environment Triggers
| Trigger | When Fired |
|---------|------------|
| `TIME_OF_DAY` | Specific time of day reached |
| `RANDOM` | Periodic random trigger |

## Auto-Discovery

When auto-discovery is enabled, the system automatically:

1. Queries the model asset for available animations
2. Identifies animations with associated sound events
3. Infers appropriate triggers based on animation names:
   - `*death*`, `*die*` → `ON_DEATH`
   - `*attack*`, `*strike*`, `*swing*` → `ON_COMBAT_START`
   - `*wave*`, `*greet*`, `*bow*` → `ON_GREETING`
   - `*talk*`, `*speak*` → `ON_DIALOGUE_LINE`
   - `*idle*`, `*fidget*` → `RANDOM`
   - Unknown patterns → `ON_INTERACT` (fallback)

## Quest Integration

The `QuestAnimationEventListener` automatically triggers NPC animations when quest events occur:

```java
// Register the listener with your quest system
QuestAnimationEventListener listener = new QuestAnimationEventListener(
    npcRegistry,
    animationService
);
questEventBus.register(listener);

// When quest events fire, NPCs automatically animate:
// - Quest accepted → ON_QUEST_ACCEPT on quest giver NPC
// - Quest completed → ON_QUEST_COMPLETE on turn-in NPC
// - Quest failed → ON_QUEST_FAIL on relevant NPC
```

## REST API (NPC Designer)

The `NPCAnimationResource` provides endpoints for the NPC Designer WebUI:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/npc/models/{modelId}/animations` | GET | Get available animations |
| `/api/npc/models/{modelId}/animation-sounds` | GET | Get animation-sound pairs |
| `/api/npc/triggers` | GET | Get all available triggers |
| `/api/npc/{npcId}/trigger/{trigger}` | POST | Test-fire a trigger |

## Best Practices

### 1. Use Auto-Discovery for Standard NPCs

```java
// Let the system discover animations from your model
NPCAnimationBehavior behavior = NPCAnimationBehavior.defaults();
```

### 2. Override Only What's Necessary

```java
// Auto-discover, but customize death animation
NPCAnimationBehavior behavior = NPCAnimationBehavior.builder()
    .autoDiscoverAnimations(true)
    .onDeath("anim:epic_death", "sfx:dramatic_death")
    .build();
```

### 3. Use Cooldowns for Frequent Triggers

```java
// Prevent fidget animation spam
SynchronizedAnimationSound fidget = SynchronizedAnimationSound.animationOnly(
    "anim:fidget",
    AnimationTrigger.RANDOM
).withCooldown(5000); // 5 second minimum between plays
```

### 4. Consider Interruptibility

```java
// Death animation should not be interrupted
SynchronizedAnimationSound death = SynchronizedAnimationSound.of(
    "anim:death",
    "sfx:death",
    AnimationTrigger.ON_DEATH
).nonInterruptible();
```

### 5. Use Pitch Variation for Natural Sound

```java
// ±15% pitch variation for more natural audio
SynchronizedAnimationSound config = new SynchronizedAnimationSound(
    "anim:greet",
    "sfx:hello",
    AnimationTrigger.ON_GREETING,
    1.0f,   // full volume
    0.15f,  // 15% pitch variation
    true,
    0
);
```

## See Also

- [NPC Integration Guide](npc-integration.md)
- [Dialogue System Guide](dialogue-system.md)
- [Quest Development Guide](quest-development.md)
- [SF-NPC-041 Specification](../../00-Argonath-Specifications/SF-NPC-041-npc-audio-integration.md)
