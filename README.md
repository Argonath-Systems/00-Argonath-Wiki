# Argonath Systems Wiki

Welcome to the comprehensive documentation for the Argonath Systems framework ecosystem - a modular architecture for creating rich, quest-driven experiences in Hytale.

> **Lord of the Rings Themed Server** - Building immersive Middle-earth experiences with platform-agnostic, extensible frameworks.

---

## 📚 Documentation Structure

### 🚀 Getting Started
| Guide | Description |
|-------|-------------|
| **[Introduction](docs/getting-started/introduction.md)** | Overview of the Argonath ecosystem |
| **[Quick Start](docs/getting-started/quick-start.md)** | Get up and running in 5 minutes |
| **[Installation Guide](docs/getting-started/installation.md)** | Detailed setup instructions |
| **[Your First Quest](docs/getting-started/first-quest.md)** | Create your first quest mod |
| **[Bundles vs Modules](docs/getting-started/bundles-vs-modules.md)** | Choose your development approach |

### 🏛️ Architecture
| Document | Description |
|----------|-------------|
| **[System Overview](docs/architecture/overview.md)** | High-level architecture design |
| **[Module Reference](docs/architecture/modules.md)** | Complete module documentation |
| **[Core Concepts](docs/architecture/concepts.md)** | Key concepts and terminology |
| **[Dependencies](docs/architecture/dependencies.md)** | Module dependencies and graphs |

### 📖 Guides

#### Quest & Objective System
| Guide | Description |
|-------|-------------|
| **[Quest Development](docs/guides/quest-development.md)** | Complete quest creation guide with objectives, conditions, and rewards |
| **[First Dungeon](docs/guides/first-dungeon.md)** | Step-by-step dungeon creation tutorial |

#### NPC & Dialogue System
| Guide | Description |
|-------|-------------|
| **[NPC Integration](docs/guides/npc-integration.md)** | Working with NPCs, behaviors, and spawning |
| **[NPC Tutorials](docs/guides/npc-tutorials.md)** | Step-by-step NPC creation tutorials |
| **[Dialogue System](docs/guides/dialogue-system.md)** | Complete dialogue tree, conditions, and quest integration |
| **[Multi-Dimensional NPC Narrative](docs/guides/multi-dimensional-npc-narrative-system.md)** | AI-driven dynamic NPC storytelling |

#### Combat System
| Guide | Description |
|-------|-------------|
| **[Combat Framework Usage](docs/guides/combat-framework-usage.md)** | Implementing combat mechanics |
| **[Combat Framework Migration](docs/guides/combat-framework-migration.md)** | Migrating from legacy combat systems |

#### UI Development
| Guide | Description |
|-------|-------------|
| **[UI Development](docs/guides/ui-development.md)** | Creating custom UIs with HyUI/HYUIML |

#### Platform & Integration
| Guide | Description |
|-------|-------------|
| **[Hytale Adapter Integration](docs/guides/hytale-adapter-integration.md)** | Accessor Pattern, event translation, and Hytale communication |
| **[Configuration](docs/guides/configuration.md)** | Configuration system guide |
| **[Storage & Persistence](docs/guides/storage.md)** | Data persistence management |

### 📘 API Reference

#### Core APIs
| Reference | Description |
|-----------|-------------|
| **[Platform Core](docs/api/platform-core.md)** | Core platform utilities and abstractions |
| **[Event System](docs/api/events.md)** | Event handling, dispatching, and listeners |

#### Framework APIs
| Reference | Description |
|-----------|-------------|
| **[Quest Framework](docs/api/framework-quest.md)** | Complete quest system APIs |
| **[NPC Framework](docs/api/framework-npc.md)** | NPC, dialogue, and behavior APIs |
| **[UI Framework](docs/api/framework-ui.md)** | User interface framework APIs |
| **[Combat Framework](docs/api/framework-combat.md)** | Combat system APIs |

#### World & Generation APIs
| Reference | Description |
|-----------|-------------|
| **[World Generation](docs/api/world-generation.md)** | Procedural world generation APIs |
| **[Dungeon Generation](docs/api/dungeon-generation.md)** | Dungeon layout and spawning APIs |

#### Legacy API Reference
| Reference | Description |
|-----------|-------------|
| **[NPC Framework (Legacy)](docs/api-reference/npc-framework.md)** | Original NPC framework documentation |

### 🔗 Integration
| Document | Description |
|----------|-------------|
| **[Module Integration](docs/integration/modules.md)** | Combining multiple frameworks |

### 🔬 Advanced Topics
| Document | Description |
|----------|-------------|
| **[Custom Conditions](docs/advanced/custom-conditions.md)** | Creating custom condition types |
| **[Custom Objectives](docs/advanced/custom-objectives.md)** | Implementing custom objectives |
| **[Performance Optimization](docs/advanced/performance.md)** | Best practices and tuning |
| **[Testing Strategies](docs/advanced/testing.md)** | Unit, integration, and E2E testing |

### 🔄 Migration & Upgrade
| Document | Description |
|----------|-------------|
| **[Migration Overview](docs/migration/overview.md)** | Migrating from legacy systems |
| **[Upgrading Versions](docs/migration/upgrading.md)** | Version upgrade procedures |
| **[Breaking Changes](docs/migration/breaking-changes.md)** | Complete changelog |

### 🤝 Contributing
| Document | Description |
|----------|-------------|
| **[Contributing Guide](CONTRIBUTING.md)** | How to contribute to the project |

---

## 🏗️ Module Overview

The Argonath Systems architecture follows a **layered approach** with strict separation of concerns. Business logic remains **platform-agnostic** through the [Accessor Pattern](docs/guides/hytale-adapter-integration.md).

### Platform Layer (01-xx)
| Module | Description |
|--------|-------------|
| **platform-core** | Foundation utilities, abstractions, and DataValue type system |
| **platform-sdk** | Development kit for extending the platform |

### Adapter Layer (02-adapter-xx)
| Module | Description |
|--------|-------------|
| **adapter-hytale** | Hytale game integration (ONLY place for `hytale.*` imports) |
| **adapter-mod-api** | Generic mod API bridge |

### Core Framework Layer (02-framework-xx)
| Module | Description |
|--------|-------------|
| **framework-accessor** | Data access patterns and accessor interfaces |
| **framework-core** | Core framework services and utilities |

### Service Framework Layer (03-framework-xx)
| Module | Description |
|--------|-------------|
| **framework-config** | Configuration management and YAML support |
| **framework-storage** | Data persistence and storage backends |
| **framework-text-styling** | Text formatting, colors, and styling |
| **framework-webserver** | Embedded web server for tools integration |

### Feature Framework Layer (04-framework-xx)
| Module | Description |
|--------|-------------|
| **framework-condition** | Condition evaluation system |
| **framework-currency** | In-game currency management |
| **framework-npc** | NPC management, dialogue, and behaviors |
| **framework-objective** | Quest objective tracking system |
| **framework-stats** | Player statistics and progression |
| **framework-worldgen** | Procedural world generation |

### High-Level Framework Layer (05-framework-xx)
| Module | Description |
|--------|-------------|
| **framework-quest** | Complete quest system with lifecycle management |
| **framework-ui** | User interface framework (HyUI/HYUIML integration) |

### Mod Layer (06-mod-xx)
| Module | Description |
|--------|-------------|
| **mod-character-races** | Character race system for Middle-earth |
| **mod-combat** | Combat mechanics and damage system |
| **mod-dungeons-raids** | Dungeon and raid content |
| **mod-guilds** | Player guild system |
| **mod-housing** | Player housing system |
| **mod-loot-tables** | Loot table definitions |
| **mod-mounts** | Mount system |
| **mod-quest-tracker** | Quest tracking UI mod |

### Tool Layer (07-tools-xx)
| Tool | Description |
|------|-------------|
| **tools-prefab-designer** | Visual prefab creation tool |
| **tools-quest-designer** | Visual quest creation tool (HyQuest) |

### Library Layer (08-lib-xx)
| Library | Description |
|---------|-------------|
| **lib-hyquest-api-client** | API client for HyQuest web service |
| **lib-prefab-api-client** | API client for Prefab service |

### Bundles
| Bundle | Description |
|--------|-------------|
| **bundle-core** | Core functionality bundle |
| **bundle-quest** | Complete quest system bundle (includes core) |

---

## 🎯 Key Concepts

### Accessor Pattern
All business logic (Frameworks, Mods) is **platform-agnostic**. Hytale API usage is isolated to the adapter layer. See [Hytale Adapter Integration](docs/guides/hytale-adapter-integration.md) for details.

```
┌─────────────────────────────────────────────────────────────┐
│  Mod Layer (06-xx)         │ Uses only Framework APIs      │
├─────────────────────────────────────────────────────────────┤
│  Framework Layer (02-05)   │ Platform-agnostic logic       │
├─────────────────────────────────────────────────────────────┤
│  Adapter Layer (02-adapter)│ Translates to/from Hytale API │
├─────────────────────────────────────────────────────────────┤
│  Hytale Game Engine        │ Native game code              │
└─────────────────────────────────────────────────────────────┘
```

### DataValue Type System (v2.0.0)
Type-safe data handling using sealed interfaces instead of raw `Object` types. Used throughout conditions, objectives, and configuration.

---

## 🚀 Quick Links

| Resource | Link |
|----------|------|
| Main Repository | [GitHub](https://github.com/yourusername/Argonath-Systems) |
| Issue Tracker | [Issues](https://github.com/yourusername/Argonath-Systems/issues) |
| Discussions | [GitHub Discussions](https://github.com/yourusername/Argonath-Systems/discussions) |
| Discord Community | [Join Discord](#) |
| Hytale Modding Docs | [hytalemodding.dev](https://hytalemodding.dev/en/docs) |

---

## 📦 Bundles vs Individual Modules

**New to Argonath?** Start with bundles:
- `bundle-core` - Everything needed for basic mod development
- `bundle-quest` - Complete quest system (includes core)

**Experienced developer?** Pick individual modules for fine-grained control.

See [Bundles vs Modules Guide](docs/getting-started/bundles-vs-modules.md) for details.

---

## 📄 License

This documentation is licensed under [CC BY 4.0](LICENSE).  
Code examples are licensed under the same license as the main project.

---

## 🤝 Support

| Channel | Purpose |
|---------|---------|
| 📖 [FAQ](docs/faq.md) | Common questions and answers |
| 💬 [Discussions](https://github.com/yourusername/Argonath-Systems/discussions) | Ask questions |
| 🐛 [Issues](https://github.com/yourusername/Argonath-Systems/issues) | Report bugs |
| 💡 [Ideas](https://github.com/yourusername/Argonath-Systems/discussions/categories/ideas) | Suggest features |

---

**Latest Update:** January 2025 | **Version:** 2.0.0
