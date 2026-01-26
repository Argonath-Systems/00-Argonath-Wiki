# System Architecture Overview

## Architectural Principles

Argonath Systems is built on **Argonath Systems** architecture, emphasizing:

1. **Layered Design** - Clear separation between platform, adapters, frameworks, and mods
2. **Dependency Inversion** - Depend on abstractions, not implementations
3. **Modular Composition** - Mix and match components as needed
4. **Single Responsibility** - Each module has one clear purpose

## The Four Layers

```
┌─────────────────────────────────────────────┐
│           MOD LAYER (06-xx)                 │
│  End-user content and gameplay features     │
│  • mod-quest-tracker                        │
└─────────────────────────────────────────────┘
                    ▲
                    │
┌─────────────────────────────────────────────┐
│      FRAMEWORK LAYER (02-05-xx)             │
│  Feature-specific implementations           │
│  • framework-quest    • framework-npc       │
│  • framework-ui       • framework-config    │
│  • framework-storage  • framework-objective │
└─────────────────────────────────────────────┘
                    ▲
                    │
┌─────────────────────────────────────────────┐
│       ADAPTER LAYER (02-adapter-xx)         │
│  Game-specific integration bridges          │
│  • adapter-hytale                           │
│  • adapter-mod-api                          │
└─────────────────────────────────────────────┘
                    ▲
                    │
┌─────────────────────────────────────────────┐
│       PLATFORM LAYER (01-xx)                │
│  Game-agnostic foundation utilities         │
│  • platform-core                            │
│  • platform-sdk                             │
└─────────────────────────────────────────────┘
```

## Layer Details

### Platform Layer (01-xx)

**Purpose**: Provide game-agnostic foundation services

**Modules**:
- `platform-core` - Core utilities, abstractions, service registry
- `platform-sdk` - Development tools and extensions

**Key Responsibilities**:
- Service registry and dependency injection
- Event bus implementation
- Common data structures
- Logging and error handling
- Resource management

**Dependencies**: None (foundation layer)

### Adapter Layer (02-adapter-xx)

**Purpose**: Bridge between platform and specific game engines

**Modules**:
- `adapter-hytale` - Hytale game integration
- `adapter-mod-api` - Generic mod API bridge

**Key Responsibilities**:
- Translate game events to platform events
- Convert platform commands to game actions
- Handle game-specific data formats
- Manage game lifecycle integration

**Dependencies**: Platform Layer

### Framework Layer (02-05-xx)

**Purpose**: Implement specific features using platform services

**Organized by prefix**:
- `02-framework-xxx` - Core frameworks (accessor, core)
- `03-framework-xxx` - Configuration and data (config, storage, text-styling)
- `04-framework-xxx` - Game mechanics (condition, npc, objective)
- `05-framework-xxx` - High-level features (quest, ui)

**Key Modules**:

**Core (02-xx)**:
- `framework-accessor` - Data access patterns
- `framework-core` - Core framework services

**Data Management (03-xx)**:
- `framework-config` - Configuration system
- `framework-storage` - Persistent data storage
- `framework-text-styling` - Text formatting and localization

**Game Mechanics (04-xx)**:
- `framework-condition` - Condition evaluation (if/when logic)
- `framework-npc` - NPC management and behaviors
- `framework-objective` - Quest objective system

**High-Level Features (05-xx)**:
- `framework-quest` - Complete quest system
- `framework-ui` - User interface framework

**Dependencies**: Platform Layer, selected Adapters

### Mod Layer (06-xx)

**Purpose**: End-user content and complete features

**Modules**:
- `mod-quest-tracker` - Quest tracking UI

**Key Responsibilities**:
- Combine frameworks into complete features
- Provide user-facing functionality
- Implement specific gameplay content

**Dependencies**: Framework Layer components

## Module Numbering System

The `XX-` prefix indicates dependency tier:

- `01-` Platform (no dependencies on other Argonath modules)
- `02-` Adapters and low-level frameworks
- `03-` Mid-level frameworks (data/config)
- `04-` High-level frameworks (game mechanics)
- `05-` Complex frameworks (quest, ui)
- `06-` Mods (complete features)

Higher numbers can depend on lower numbers, never the reverse.

## Dependency Flow

```
┌──────────────┐
│ Quest Mod    │ (06)
└──────┬───────┘
       │ uses
       ▼
┌──────────────┐     ┌──────────────┐
│framework-    │────▶│framework-    │ (05)
│quest         │     │ui            │
└──────┬───────┘     └──────┬───────┘
       │                    │
       │ uses               │ uses
       ▼                    ▼
┌──────────────┐     ┌──────────────┐
│framework-    │     │framework-    │ (04)
│objective     │     │condition     │
└──────┬───────┘     └──────┬───────┘
       │                    │
       └─────────┬──────────┘
                 │ uses
                 ▼
┌─────────────────────────────────────┐
│framework-config / framework-storage │ (03)
└─────────────┬───────────────────────┘
              │ uses
              ▼
┌─────────────────────────────────────┐
│     framework-core / accessor       │ (02)
└─────────────┬───────────────────────┘
              │ uses
              ▼
┌─────────────────────────────────────┐
│         platform-core               │ (01)
└─────────────────────────────────────┘
```

## Bundles

**Purpose**: Pre-configured module collections for common use cases

**Available Bundles**:
- `bundle-core` - Essential platform + core frameworks
- `bundle-quest` - Complete quest system (includes core)

**Bundle Composition**:

```
bundle-quest includes:
├── platform-core
├── framework-core
├── framework-config
├── framework-storage
├── framework-condition
├── framework-objective
├── framework-quest
└── adapter-hytale
```

Bundles are convenience packages - you can always use individual modules instead.

## Cross-Cutting Concerns

### Event System
- Defined in `platform-core`
- Used by all frameworks
- Adapters translate game events

### Configuration
- `framework-config` provides system
- Each framework can define config schemas
- Centralized or distributed configs

### Storage
- `framework-storage` provides persistence
- Frameworks define data models
- Pluggable storage backends

### Conditions
- `framework-condition` provides evaluation
- Other frameworks register condition types
- Used for quest logic, NPC dialogues, etc.

## Extension Points

Every framework provides extension points:

1. **Service Interfaces** - Implement to add behavior
2. **Registry Systems** - Register custom types
3. **Event Listeners** - React to framework events
4. **Configuration Schemas** - Define custom configs

See [Advanced Topics](../advanced/) for extension guides.

## Design Patterns

### Service Registry Pattern
Central registry for dependency injection and service location.

### Event-Driven Architecture
Loose coupling through event bus.

### Factory Pattern
Builders and factories for object creation.

### Strategy Pattern
Pluggable implementations (storage backends, condition evaluators).

### Observer Pattern
Event listeners and callbacks.

## Next Steps

- **[Module Structure](modules.md)** - Detailed module breakdown
- **[Core Concepts](concepts.md)** - Key terminology and patterns
- **[Dependency Graph](dependencies.md)** - Visual dependency map
- **[Quest Development Guide](../guides/quest-development.md)** - Apply this architecture

---

**Understanding the architecture?** You're ready to start building! Head to [Quest Development Guide](../guides/quest-development.md).
