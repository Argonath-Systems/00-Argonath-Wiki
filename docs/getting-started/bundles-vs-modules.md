# Bundles vs Modules

Understanding when to use pre-built bundles versus individual modules.

## Overview

Argonath Systems offers two distribution strategies:

| Approach | Best For | Flexibility | Complexity |
|----------|----------|-------------|------------|
| **Bundles** | End users, quick start | Low | Low |
| **Modules** | Developers, custom builds | High | Medium |

## Bundles

### What Are Bundles?

Pre-configured collections of modules packaged as single JAR files, ready to use.

### Available Bundles

#### 1. Quest Bundle (`bundle-quest`)

**Purpose:** Complete quest system with all dependencies

**Includes:**
- Platform Core & SDK
- Hytale Adapter & Mod API Adapter
- Framework Core, Accessor, Config, Storage, Text Styling
- Quest Framework
- NPC Framework
- Objective Framework
- Condition Framework
- Quest Tracker Mod

**Use When:**
- Building quest-based mods
- Need full quest system
- Want quick setup
- Don't need customization

**Example:**
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-quest:1.0.0'
}
```

#### 2. Core Bundle (`bundle-core`)

**Purpose:** Minimal platform setup for custom systems

**Includes:**
- Platform Core & SDK
- Hytale Adapter & Mod API Adapter
- Framework Core, Accessor, Config, Storage

**Use When:**
- Building custom frameworks
- Need platform layer only
- Maximum control over dependencies
- Minimal footprint

**Example:**
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-core:1.0.0'
}
```

### Bundle Advantages

✅ **Quick Setup** - Single dependency, everything works
✅ **Tested Together** - Pre-validated compatibility
✅ **Easy Updates** - Update one version number
✅ **Beginner Friendly** - No dependency management
✅ **Smaller Distribution** - One JAR vs multiple

### Bundle Disadvantages

❌ **Larger Size** - Includes modules you may not need
❌ **Less Control** - Can't exclude specific modules
❌ **Version Lock** - All modules same version
❌ **Update Overhead** - Can't update individual parts

### When to Use Bundles

**✅ Use Bundles If:**
- You're new to Argonath Systems
- Building a quest mod
- Want minimal configuration
- Need everything pre-configured
- Targeting end users (players)
- Prototyping quickly

**❌ Don't Use Bundles If:**
- Need specific module versions
- Building library/framework
- Want minimal dependencies
- Need fine-grained control
- Publishing to Maven

## Individual Modules

### What Are Modules?

Separate, independent libraries that can be mixed and matched.

### Module Categories

#### Platform Layer (Required)
```gradle
implementation 'com.hytale.argonath:platform-core:1.0.0'
implementation 'com.hytale.argonath:platform-sdk:1.0.0'
```

**Always needed for any Argonath mod**

#### Adapters (Pick One or Both)
```gradle
implementation 'com.hytale.argonath:adapter-hytale:1.0.0'      // Hytale game
implementation 'com.hytale.argonath:adapter-mod-api:1.0.0'     // Mod API
```

**Connects platform to game engine**

#### Core Frameworks (Common)
```gradle
implementation 'com.hytale.argonath:framework-core:1.0.0'
implementation 'com.hytale.argonath:framework-accessor:1.0.0'
implementation 'com.hytale.argonath:framework-config:1.0.0'
implementation 'com.hytale.argonath:framework-storage:1.0.0'
implementation 'com.hytale.argonath:framework-text-styling:1.0.0'
```

**Foundational utilities used by other frameworks**

#### Feature Frameworks (Optional)
```gradle
// Quest System
implementation 'com.hytale.argonath:framework-quest:1.0.0'
implementation 'com.hytale.argonath:framework-npc:1.0.0'
implementation 'com.hytale.argonath:framework-objective:1.0.0'
implementation 'com.hytale.argonath:framework-condition:1.0.0'

// UI System
implementation 'com.hytale.argonath:framework-ui:1.0.0'
```

**Add only what you need**

### Module Advantages

✅ **Granular Control** - Choose exactly what you need
✅ **Smaller Footprint** - Only include necessary code
✅ **Version Flexibility** - Mix versions if needed
✅ **Dependency Clarity** - Explicit about what you use
✅ **Better for Libraries** - Don't force dependencies on users

### Module Disadvantages

❌ **Complex Setup** - Must manage dependencies manually
❌ **Version Conflicts** - Easy to get incompatible versions
❌ **More Configuration** - Multiple dependency declarations
❌ **Harder Debugging** - More places for issues

### When to Use Modules

**✅ Use Modules If:**
- Building a library/framework
- Need specific versions
- Want minimal dependencies
- Publishing to Maven/repository
- Know exactly what you need
- Building custom systems

**❌ Don't Use Modules If:**
- New to the framework
- Want quick prototyping
- Don't understand dependencies
- Building simple quest mod

## Decision Tree

```
Are you building a quest mod?
├─ YES → Use Quest Bundle
└─ NO
   ├─ Do you need UI framework only?
   │  ├─ YES → Core Bundle + framework-ui module
   │  └─ NO
   │     └─ Building custom framework?
   │        ├─ YES → Use individual modules
   │        └─ NO → Use Core Bundle
```

## Common Scenarios

### Scenario 1: Simple Quest Mod

**Goal:** Add quests to Hytale

**Solution:** Quest Bundle
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-quest:1.0.0'
}
```

**Why:** Everything included, no thinking required.

### Scenario 2: Custom Economy System

**Goal:** Build economy mod, no quests

**Solution:** Core Bundle + Storage
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-core:1.0.0'
    // Storage already in core bundle
}
```

**Why:** Don't need quest frameworks, just platform.

### Scenario 3: NPC Dialog System (No Quests)

**Goal:** Advanced NPC conversations only

**Solution:** Individual Modules
```gradle
dependencies {
    // Platform
    implementation 'com.hytale.argonath:platform-core:1.0.0'
    implementation 'com.hytale.argonath:platform-sdk:1.0.0'
    
    // Adapter
    implementation 'com.hytale.argonath:adapter-hytale:1.0.0'
    
    // Minimal frameworks
    implementation 'com.hytale.argonath:framework-core:1.0.0'
    implementation 'com.hytale.argonath:framework-npc:1.0.0'
}
```

**Why:** Only need NPC framework, not entire quest system.

### Scenario 4: Building a Library

**Goal:** Create reusable utilities for other mods

**Solution:** Individual Platform Modules Only
```gradle
dependencies {
    compileOnly 'com.hytale.argonath:platform-core:1.0.0'
    compileOnly 'com.hytale.argonath:platform-sdk:1.0.0'
}
```

**Why:** `compileOnly` - don't force dependencies on users.

### Scenario 5: Rapid Prototyping

**Goal:** Test an idea quickly

**Solution:** Quest Bundle (even if not using quests)
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-quest:1.0.0'
}
```

**Why:** Get everything, figure out what you need later.

### Scenario 6: Production Mod

**Goal:** Published mod with specific requirements

**Solution:** Carefully Selected Modules
```gradle
dependencies {
    // Exactly what we need, nothing more
    implementation 'com.hytale.argonath:platform-core:1.0.0'
    implementation 'com.hytale.argonath:platform-sdk:1.0.0'
    implementation 'com.hytale.argonath:adapter-hytale:1.0.0'
    implementation 'com.hytale.argonath:framework-core:1.0.0'
    implementation 'com.hytale.argonath:framework-config:1.0.0'
    implementation 'com.hytale.argonath:framework-ui:1.0.0'
}
```

**Why:** Minimal size, clear dependencies, professional.

## Mixing Bundles and Modules

### Can You Mix Them?

**Generally NO** - causes duplicate classes and conflicts.

**Exception:** Add modules NOT in your bundle
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-core:1.0.0'
    // Add UI framework not in core bundle
    implementation 'com.hytale.argonath:framework-ui:1.0.0'  // ✅ OK
}
```

**Never Do:**
```gradle
dependencies {
    implementation 'com.hytale.argonath:bundle-quest:1.0.0'
    implementation 'com.hytale.argonath:framework-quest:1.0.0'  // ❌ DUPLICATE
}
```

## Migration Between Approaches

### From Bundle to Modules

1. **Identify what you're actually using**
   ```java
   // Scan imports
   import com.hytale.argonath.framework.quest.*;  // Using quest
   import com.hytale.argonath.framework.ui.*;     // Using UI
   ```

2. **Replace bundle with specific modules**
   ```gradle
   dependencies {
       // Replace this:
       // implementation 'com.hytale.argonath:bundle-quest:1.0.0'
       
       // With these:
       implementation 'com.hytale.argonath:platform-core:1.0.0'
       implementation 'com.hytale.argonath:platform-sdk:1.0.0'
       implementation 'com.hytale.argonath:adapter-hytale:1.0.0'
       implementation 'com.hytale.argonath:framework-core:1.0.0'
       implementation 'com.hytale.argonath:framework-quest:1.0.0'
       implementation 'com.hytale.argonath:framework-ui:1.0.0'
   }
   ```

3. **Test and verify**

### From Modules to Bundle

1. **Determine if bundle covers your needs**
2. **Replace all individual dependencies with bundle**
   ```gradle
   dependencies {
       // Replace 10+ lines with:
       implementation 'com.hytale.argonath:bundle-quest:1.0.0'
   }
   ```
3. **Remove unused imports/code**

## Best Practices

### For Bundles

✅ **DO:**
- Use for learning and prototyping
- Stick to one bundle per project
- Update all at once
- Document which bundle you use

❌ **DON'T:**
- Mix multiple bundles
- Mix bundles with duplicate modules
- Cherry-pick from bundles

### For Modules

✅ **DO:**
- Keep versions consistent
- Document dependency tree
- Use dependency management
- Understand what each does

❌ **DON'T:**
- Mix major versions (1.x with 2.x)
- Add modules you don't use
- Forget transitive dependencies

## Version Management

### Bundle Versioning

```gradle
// All modules in bundle share version
implementation 'com.hytale.argonath:bundle-quest:1.0.0'
// Internally: platform-core:1.0.0, framework-quest:1.0.0, etc.
```

### Module Versioning

```gradle
// Can use different versions (not recommended)
implementation 'com.hytale.argonath:platform-core:1.0.0'
implementation 'com.hytale.argonath:framework-quest:1.1.0'  // Newer version
```

**Recommendation:** Keep all Argonath modules at same version.

## Performance Considerations

### JAR Size Comparison

| Approach | Typical Size | Load Time |
|----------|-------------|-----------|
| Quest Bundle | ~5-8 MB | Fast |
| Core Bundle | ~2-3 MB | Faster |
| Minimal Modules | ~500 KB - 2 MB | Fastest |

### Runtime Performance

**No difference** - unused classes not instantiated regardless of approach.

## Summary

| Factor | Bundles | Modules |
|--------|---------|---------|
| **Ease of Use** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Flexibility** | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Size** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Beginner Friendly** | ⭐⭐⭐⭐⭐ | ⭐⭐ |
| **Library Development** | ⭐ | ⭐⭐⭐⭐⭐ |
| **Maintenance** | ⭐⭐⭐⭐ | ⭐⭐⭐ |

### Quick Recommendations

- **New to Argonath?** → Use Quest Bundle
- **Building quest mod?** → Use Quest Bundle
- **Building custom framework?** → Use Modules
- **Need UI only?** → Core Bundle + framework-ui
- **Publishing library?** → Use Modules with `compileOnly`
- **Not sure?** → Start with bundle, migrate later if needed

---

**Still confused?** Ask in [Discussions](https://github.com/YOUR_USERNAME/Argonath-Systems/discussions)!
