# Installation Guide

Complete installation instructions for Argonath Systems framework ecosystem.

## Prerequisites

### Required
- **Hytale** - Installed and running
- **Java Development Kit (JDK)** - Version 17 or higher
- **Gradle** - Version 7.0+ (or use wrapper)

### Recommended
- **Git** - For version control and cloning repositories
- **IntelliJ IDEA** or **Eclipse** - For development
- **Hytale Mod Loader** - Latest version

## Installation Methods

### Method 1: Using Bundles (Recommended for Users)

Bundles provide pre-configured collections of modules for specific use cases.

#### Quest Bundle
Includes everything needed for quest development:

1. Download the latest `bundle-quest.jar` from [Releases](https://github.com/Argonath-Systems/Argonath-Systems/releases)
2. Place in your Hytale `mods/` folder
3. Restart Hytale

**Includes:**
- Platform Core & SDK
- Hytale Adapter
- Framework Core, Config, Storage
- Quest Framework
- NPC Framework
- Objective Framework
- Condition Framework
- Quest Tracker Mod

#### Core Bundle
Minimal setup for building custom systems:

1. Download `bundle-core.jar`
2. Place in `mods/` folder
3. Restart Hytale

**Includes:**
- Platform Core & SDK
- Hytale Adapter
- Framework Core, Config, Storage

### Method 2: Individual Modules (For Developers)

Install only the modules you need for maximum control.

#### Step 1: Add Repository

Add to your `build.gradle`:

```gradle
repositories {
    maven {
        url = 'https://maven.pkg.github.com/Argonath-Systems/Argonath-Systems'
        credentials {
            username = project.findProperty("gpr.user") ?: System.getenv("GITHUB_ACTOR")
            password = project.findProperty("gpr.key") ?: System.getenv("GITHUB_TOKEN")
        }
    }
}
```

#### Step 2: Add Dependencies

```gradle
dependencies {
    // Required: Platform Layer
    implementation 'com.hytale.argonath:platform-core:1.0.0'
    implementation 'com.hytale.argonath:platform-sdk:1.0.0'
    
    // Required: Hytale Integration
    implementation 'com.hytale.argonath:adapter-hytale:1.0.0'
    
    // Core Frameworks (pick what you need)
    implementation 'com.hytale.argonath:framework-core:1.0.0'
    implementation 'com.hytale.argonath:framework-config:1.0.0'
    implementation 'com.hytale.argonath:framework-storage:1.0.0'
    
    // Quest System
    implementation 'com.hytale.argonath:framework-quest:1.0.0'
    implementation 'com.hytale.argonath:framework-npc:1.0.0'
    implementation 'com.hytale.argonath:framework-objective:1.0.0'
    implementation 'com.hytale.argonath:framework-condition:1.0.0'
    
    // UI System
    implementation 'com.hytale.argonath:framework-ui:1.0.0'
}
```

#### Step 3: Build and Run

```bash
./gradlew build
./gradlew runClient
```

### Method 3: Development Build (From Source)

For contributors or custom modifications:

#### Step 1: Clone Repository

```bash
git clone https://github.com/Argonath-Systems/Argonath-Systems.git
cd Argonath-Systems
```

#### Step 2: Install Frameworks

```bash
# Uses the installation script
./install-frameworks.sh
```

Or manually install each module:

```bash
# Navigate to each module and install
cd 01-platform-core && ./gradlew publishToMavenLocal && cd ..
cd 01-platform-sdk && ./gradlew publishToMavenLocal && cd ..
cd 02-adapter-hytale && ./gradlew publishToMavenLocal && cd ..
# ... repeat for all modules
```

#### Step 3: Build Bundles

```bash
# Build both bundles
./gradlew :bundle-core:build :bundle-quest:build

# Or use just commands
just build-all
```

Bundles will be in:
- `bundle-core/build/libs/bundle-core-1.0.0.jar`
- `bundle-quest/build/libs/bundle-quest-1.0.0.jar`

#### Step 4: Test

```bash
# Copy to Hytale mods folder
cp bundle-quest/build/libs/*.jar /path/to/hytale/mods/

# Or use symbolic links for development
ln -s $(pwd)/bundle-quest/build/libs/bundle-quest-1.0.0.jar /path/to/hytale/mods/
```

## Configuration

### First-Time Setup

1. **Launch Hytale** with the bundle/modules installed
2. **Check logs** for successful initialization:
   ```
   [Argonath] Platform Core initialized
   [Argonath] Quest Framework loaded
   ```
3. **Configure** (if needed) in `config/argonath/`

### Configuration Files

Created automatically on first run:

```
config/argonath/
├── platform.json          # Platform settings
├── quest-framework.json   # Quest system config
├── storage.json           # Data persistence config
└── ui.json               # UI framework config
```

### Basic Configuration

Edit `config/argonath/platform.json`:

```json
{
  "debug": false,
  "logLevel": "INFO",
  "features": {
    "quests": true,
    "npcs": true,
    "ui": true
  }
}
```

## Verification

### Verify Installation

Run in-game command:
```
/argonath version
```

Should output:
```
Argonath Systems v1.0.0
- Platform Core: v1.0.0
- Quest Framework: v1.0.0
- NPC Framework: v1.0.0
```

### Test Quest System

```
/argonath quest list
```

Should show available quests (empty on first install).

### Check Logs

Look for successful initialization in `logs/latest.log`:

```
[INFO] Argonath Platform initialized successfully
[INFO] Quest Framework ready
[INFO] 0 quests loaded
```

## Troubleshooting

### Common Issues

#### "Module not found" error
**Cause:** Missing dependencies
**Solution:** Install bundle-core or all required platform modules

#### "Version conflict" error
**Cause:** Incompatible module versions
**Solution:** Ensure all modules are from the same version (e.g., all 1.0.0)

#### Quests not loading
**Cause:** Missing quest framework modules
**Solution:** Install bundle-quest or add framework-quest + dependencies

#### UI not appearing
**Cause:** Missing UI framework
**Solution:** Add framework-ui dependency

### Getting Help

- **GitHub Issues**: [Report bugs](https://github.com/Argonath-Systems/Argonath-Systems/issues)
- **Discord**: Join our community server
- **Discussions**: [Ask questions](https://github.com/Argonath-Systems/Argonath-Systems/discussions)

## Next Steps

- **[Create Your First Quest](first-quest.md)** - Tutorial walkthrough
- **[Understand Bundles vs Modules](bundles-vs-modules.md)** - Choose the right approach
- **[Architecture Overview](../architecture/overview.md)** - Learn the system design

## Version Compatibility

| Argonath Version | Hytale Version | Java Version |
|------------------|----------------|--------------|
| 1.0.0+           | Beta 1.0+      | 17+          |

## Update Strategy

### Updating Bundles

1. Download new version
2. Replace old JAR in `mods/` folder
3. Restart Hytale
4. Check compatibility notes in CHANGELOG

### Updating Individual Modules

Update version in `build.gradle`:

```gradle
dependencies {
    implementation 'com.hytale.argonath:platform-core:1.1.0' // updated from 1.0.0
}
```

Then rebuild:
```bash
./gradlew clean build
```

---

**Installation complete!** Ready to [create your first quest](first-quest.md).
