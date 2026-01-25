# Frequently Asked Questions

## General Questions

### What is Argonath Systems?

Argonath Systems is a modular framework ecosystem for creating quest-driven experiences in Hytale. It provides reusable components for quests, NPCs, UI, configuration, and more.

### Is it only for Hytale?

The core platform is game-agnostic. While we provide a Hytale adapter, you could create adapters for other games or platforms.

### Do I need to use all modules?

No! The modular design lets you pick only what you need. Use bundles for convenience or individual modules for control.

### Is it free?

Yes, Argonath Systems is open source under the MIT License.

## Getting Started

### What do I need to get started?

- Java 17 or higher
- Maven 3.8+
- Basic Java and Maven knowledge
- A Hytale development environment

### Which bundle should I use?

- **bundle-core**: Basic mod development
- **bundle-quest**: Complete quest system

See [Bundles vs Modules](docs/getting-started/bundles-vs-modules.md) for details.

### Where are the examples?

Check the `examples/` directory in the main repository or browse the [examples section](examples/) in this wiki.

### How do I report bugs?

Open an issue in the [main repository](https://github.com/yourusername/Argonath-Systems/issues).

## Development Questions

### Can I create custom quest objectives?

Yes! See [Custom Objectives](docs/advanced/custom-objectives.md).

### How do I persist quest data?

Use `framework-storage`. See [Storage Guide](docs/guides/storage.md).

### Can I create custom UI components?

Absolutely! See [UI Development](docs/guides/ui-development.md).

### How do I handle quest branching?

Use the condition system. See [Conditions](docs/advanced/custom-conditions.md).

## Technical Questions

### What's the difference between frameworks and bundles?

- **Frameworks**: Individual modules with specific features
- **Bundles**: Pre-configured collections of frameworks

### How are dependencies managed?

Maven handles all dependencies. Just add the bundle or modules you need to your `pom.xml`.

### Can I use this with other mods?

Yes! Argonath is designed to work alongside other mods. See [External Mods Integration](docs/integration/external-mods.md).

### What's the performance impact?

Minimal. The framework is optimized for efficiency. See [Performance Guide](docs/advanced/performance.md).

## Troubleshooting

### Build fails with "module not found"

Ensure you've added the Argonath repository to your `pom.xml`:
```xml
<repositories>
    <repository>
        <id>argonath-systems</id>
        <url>https://maven.pkg.github.com/yourusername/Argonath-Systems</url>
    </repository>
</repositories>
```

### Quest doesn't appear in game

Check:
1. Quest is registered: `questRegistry.register(quest)`
2. Prerequisites are met
3. Quest giver is placed in game
4. Debug logging is enabled

### NPCs don't respond to interaction

Verify:
1. NPC is registered: `npcRegistry.register(npc)`
2. Interaction handler is bound
3. Interaction distance is configured
4. Hytale adapter is initialized

### UI doesn't display

Check:
1. UI manager is initialized
2. UI component is created and shown
3. UI coordinates are visible on screen
4. No conflicting UI from other mods

## Migration Questions

### Can I migrate from legacy systems?

Yes! See [Migration Guide](docs/migration/overview.md).

### Will my old quests work?

Migration tools are provided. See [Upgrade Guide](docs/migration/upgrading.md).

### What about breaking changes?

All breaking changes are documented in [Breaking Changes](docs/migration/breaking-changes.md).

## Contributing

### How can I contribute?

See [Contributing Guide](CONTRIBUTING.md) for details on:
- Documentation improvements
- Code contributions
- Bug reports
- Feature requests

### Do I need permission to contribute?

No! Fork the repo and submit a PR. We welcome all contributions.

### What if my PR is rejected?

We'll provide feedback. You can revise and resubmit.

## Community

### Is there a Discord?

Coming soon! Check the README for updates.

### Where can I ask questions?

- [GitHub Discussions](https://github.com/yourusername/Argonath-Systems/discussions)
- [Discord](#) (when available)
- [Stack Overflow](https://stackoverflow.com) with tag `argonath-systems`

### Can I share my mods?

Please do! We'd love to see what you create. Share in Discussions!

---

**Don't see your question?** Ask in [Discussions](https://github.com/yourusername/Argonath-Systems/discussions) or open an [Issue](https://github.com/yourusername/Argonath-Systems/issues).
