# Custom Conditions

Advanced guide to creating custom condition types in the Argonath Systems framework.

## Table of Contents

- [Overview](#overview)
- [Condition Architecture](#condition-architecture)
- [Creating Custom Conditions](#creating-custom-conditions)
- [Implementing the Condition Interface](#implementing-the-condition-interface)
- [Registration and Discovery](#registration-and-discovery)
- [Complex Condition Logic](#complex-condition-logic)
- [Composite Conditions](#composite-conditions)
- [Performance Optimization](#performance-optimization)
- [Testing Custom Conditions](#testing-custom-conditions)
- [Best Practices](#best-practices)

## Overview

Custom conditions allow you to extend the quest system with domain-specific logic tailored to your mod's requirements. Conditions are evaluated at runtime to determine quest progression, NPC interactions, and feature availability.

### When to Create Custom Conditions

- **Domain-Specific Logic**: Business rules unique to your mod
- **Performance Critical Paths**: Optimized evaluation for high-frequency checks
- **Complex State Validation**: Multi-system state coordination
- **Reusable Logic**: Shared validation across multiple quests

## Condition Architecture

### Core Interfaces

```java
package com.argonath.framework.condition.api;

/**
 * Core interface for all conditions in the Argonath framework.
 * Conditions are immutable, serializable, and thread-safe.
 */
public interface Condition {
    /**
     * Evaluates the condition for the given context.
     * 
     * @param context The evaluation context containing player, world state, etc.
     * @return true if the condition is met, false otherwise
     */
    boolean evaluate(ConditionContext context);
    
    /**
     * Returns a unique identifier for this condition type.
     * Used for serialization and registration.
     */
    String getType();
    
    /**
     * Serializes this condition to a configuration format.
     * Must be reversible via the corresponding deserializer.
     */
    ConfigNode serialize();
    
    /**
     * Returns a human-readable description of this condition.
     * Used for debugging and UI display.
     */
    String getDescription();
    
    /**
     * Optional: Returns estimated evaluation cost (0-100).
     * Used for optimization and ordering.
     */
    default int getEvaluationCost() {
        return 50; // Medium cost by default
    }
}
```

### Condition Context

```java
package com.argonath.framework.condition.api;

import com.argonath.platform.core.player.Player;
import com.argonath.platform.core.world.World;
import java.util.Optional;

/**
 * Immutable context provided during condition evaluation.
 * Contains all information needed to evaluate conditions.
 */
public interface ConditionContext {
    /**
     * Returns the player being evaluated, if applicable.
     */
    Optional<Player> getPlayer();
    
    /**
     * Returns the world where evaluation occurs.
     */
    World getWorld();
    
    /**
     * Returns the current server time in milliseconds.
     */
    long getServerTime();
    
    /**
     * Retrieves custom context data by key.
     * Used for passing additional evaluation parameters.
     */
    <T> Optional<T> getData(String key, Class<T> type);
    
    /**
     * Creates a child context with additional data.
     * Useful for nested condition evaluation.
     */
    ConditionContext withData(String key, Object value);
}
```

## Creating Custom Conditions

### Step 1: Define Your Condition Class

```java
package com.example.mymod.conditions;

import com.argonath.framework.condition.api.Condition;
import com.argonath.framework.condition.api.ConditionContext;
import com.argonath.framework.config.api.ConfigNode;

/**
 * Condition that checks if a player has completed a specific achievement.
 */
public class AchievementCondition implements Condition {
    
    public static final String TYPE = "mymod:achievement";
    
    private final String achievementId;
    private final boolean requireCompleted;
    
    public AchievementCondition(String achievementId, boolean requireCompleted) {
        this.achievementId = Objects.requireNonNull(achievementId, "achievementId");
        this.requireCompleted = requireCompleted;
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        return context.getPlayer()
            .map(player -> {
                AchievementTracker tracker = getAchievementTracker(player);
                boolean hasAchievement = tracker.hasCompleted(achievementId);
                return requireCompleted ? hasAchievement : !hasAchievement;
            })
            .orElse(false); // No player context = condition fails
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public ConfigNode serialize() {
        ConfigNode node = ConfigNode.root();
        node.node("type").set(TYPE);
        node.node("achievement").set(achievementId);
        node.node("completed").set(requireCompleted);
        return node;
    }
    
    @Override
    public String getDescription() {
        String status = requireCompleted ? "completed" : "not completed";
        return String.format("Player has %s achievement '%s'", status, achievementId);
    }
    
    @Override
    public int getEvaluationCost() {
        return 30; // Low cost - simple lookup
    }
    
    private AchievementTracker getAchievementTracker(Player player) {
        // Integration with your achievement system
        return AchievementSystem.getInstance().getTracker(player.getUUID());
    }
}
```

### Step 2: Create a Deserializer

```java
package com.example.mymod.conditions;

import com.argonath.framework.condition.api.ConditionDeserializer;
import com.argonath.framework.config.api.ConfigNode;

/**
 * Deserializer for AchievementCondition.
 */
public class AchievementConditionDeserializer implements ConditionDeserializer {
    
    @Override
    public String getType() {
        return AchievementCondition.TYPE;
    }
    
    @Override
    public Condition deserialize(ConfigNode node) throws DeserializationException {
        String achievementId = node.node("achievement")
            .getString()
            .orElseThrow(() -> new DeserializationException("Missing 'achievement' field"));
            
        boolean completed = node.node("completed")
            .getBoolean()
            .orElse(true); // Default to requiring completion
            
        return new AchievementCondition(achievementId, completed);
    }
    
    @Override
    public void validate(ConfigNode node) throws ValidationException {
        if (!node.node("achievement").isPresent()) {
            throw new ValidationException("Achievement ID is required");
        }
        
        String achievementId = node.node("achievement").getString().get();
        if (!AchievementRegistry.isValidId(achievementId)) {
            throw new ValidationException("Unknown achievement: " + achievementId);
        }
    }
}
```

## Implementing the Condition Interface

### Advanced Implementation Pattern

```java
package com.example.mymod.conditions;

import com.argonath.framework.condition.api.Condition;
import com.argonath.framework.condition.api.ConditionContext;
import com.argonath.framework.config.api.ConfigNode;
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import java.util.concurrent.TimeUnit;

/**
 * Complex condition that checks player reputation with a faction.
 * Demonstrates caching and performance optimization.
 */
public class ReputationCondition implements Condition {
    
    public static final String TYPE = "mymod:reputation";
    
    // Cache for reputation lookups (reduces database queries)
    private static final Cache<String, Integer> REPUTATION_CACHE = CacheBuilder.newBuilder()
        .expireAfterWrite(30, TimeUnit.SECONDS)
        .maximumSize(1000)
        .build();
    
    private final String factionId;
    private final ComparisonOperator operator;
    private final int threshold;
    
    public ReputationCondition(String factionId, ComparisonOperator operator, int threshold) {
        this.factionId = Objects.requireNonNull(factionId, "factionId");
        this.operator = Objects.requireNonNull(operator, "operator");
        this.threshold = threshold;
        
        // Validate threshold range
        if (threshold < -100 || threshold > 100) {
            throw new IllegalArgumentException("Reputation threshold must be between -100 and 100");
        }
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        return context.getPlayer()
            .map(player -> {
                int reputation = getReputation(player.getUUID(), factionId);
                return operator.compare(reputation, threshold);
            })
            .orElse(false);
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public ConfigNode serialize() {
        ConfigNode node = ConfigNode.root();
        node.node("type").set(TYPE);
        node.node("faction").set(factionId);
        node.node("operator").set(operator.name());
        node.node("threshold").set(threshold);
        return node;
    }
    
    @Override
    public String getDescription() {
        return String.format("Reputation with '%s' %s %d", 
            factionId, operator.getSymbol(), threshold);
    }
    
    @Override
    public int getEvaluationCost() {
        return 40; // Medium-low cost with caching
    }
    
    private int getReputation(UUID playerId, String faction) {
        String cacheKey = playerId + ":" + faction;
        
        Integer cached = REPUTATION_CACHE.getIfPresent(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // Fetch from storage (potentially expensive)
        int reputation = ReputationSystem.getInstance()
            .getReputation(playerId, faction);
            
        REPUTATION_CACHE.put(cacheKey, reputation);
        return reputation;
    }
    
    /**
     * Comparison operators for numeric conditions.
     */
    public enum ComparisonOperator {
        EQUAL("=") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual == expected;
            }
        },
        NOT_EQUAL("≠") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual != expected;
            }
        },
        GREATER_THAN(">") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual > expected;
            }
        },
        GREATER_THAN_OR_EQUAL("≥") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual >= expected;
            }
        },
        LESS_THAN("<") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual < expected;
            }
        },
        LESS_THAN_OR_EQUAL("≤") {
            @Override
            public boolean compare(int actual, int expected) {
                return actual <= expected;
            }
        };
        
        private final String symbol;
        
        ComparisonOperator(String symbol) {
            this.symbol = symbol;
        }
        
        public String getSymbol() {
            return symbol;
        }
        
        public abstract boolean compare(int actual, int expected);
    }
}
```

## Registration and Discovery

### Manual Registration

```java
package com.example.mymod;

import com.argonath.framework.condition.api.ConditionRegistry;
import com.argonath.platform.sdk.ModInitializer;

public class MyMod implements ModInitializer {
    
    @Override
    public void onInitialize() {
        // Register condition deserializers
        ConditionRegistry registry = ConditionRegistry.getInstance();
        
        registry.register(new AchievementConditionDeserializer());
        registry.register(new ReputationConditionDeserializer());
        registry.register(new TimeRangeConditionDeserializer());
        registry.register(new WeatherConditionDeserializer());
        
        getLogger().info("Registered {} custom conditions", 4);
    }
}
```

### Auto-Discovery via Annotations

```java
package com.example.mymod.conditions;

import com.argonath.framework.condition.api.RegisterCondition;

/**
 * Condition that uses annotation-based registration.
 */
@RegisterCondition(
    type = "mymod:time_range",
    description = "Checks if current time is within a range"
)
public class TimeRangeCondition implements Condition {
    
    private final int startHour;
    private final int endHour;
    
    // Implementation...
    
    /**
     * Factory method for annotation-based deserialization.
     */
    @ConditionFactory
    public static TimeRangeCondition create(ConfigNode node) {
        int start = node.node("start").getInt().orElse(0);
        int end = node.node("end").getInt().orElse(24);
        return new TimeRangeCondition(start, end);
    }
}
```

## Complex Condition Logic

### State-Dependent Conditions

```java
package com.example.mymod.conditions;

/**
 * Condition that checks multiple quest states.
 * Useful for quest chains with complex dependencies.
 */
public class QuestChainCondition implements Condition {
    
    public static final String TYPE = "mymod:quest_chain";
    
    private final List<QuestRequirement> requirements;
    private final ChainLogic logic;
    
    public QuestChainCondition(List<QuestRequirement> requirements, ChainLogic logic) {
        this.requirements = List.copyOf(requirements);
        this.logic = logic;
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        return context.getPlayer()
            .map(player -> {
                QuestManager questManager = getQuestManager();
                
                List<Boolean> results = requirements.stream()
                    .map(req -> evaluateRequirement(player, req, questManager))
                    .collect(Collectors.toList());
                    
                return logic.evaluate(results);
            })
            .orElse(false);
    }
    
    private boolean evaluateRequirement(Player player, QuestRequirement req, 
                                       QuestManager manager) {
        Optional<QuestProgress> progress = manager.getProgress(player, req.questId);
        
        switch (req.state) {
            case COMPLETED:
                return progress.map(QuestProgress::isCompleted).orElse(false);
            case IN_PROGRESS:
                return progress.map(p -> !p.isCompleted()).orElse(false);
            case NOT_STARTED:
                return progress.isEmpty();
            case ANY:
                return true;
            default:
                return false;
        }
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public int getEvaluationCost() {
        return 60 + (requirements.size() * 10); // Higher cost for more requirements
    }
    
    public static class QuestRequirement {
        public final String questId;
        public final QuestState state;
        
        public QuestRequirement(String questId, QuestState state) {
            this.questId = questId;
            this.state = state;
        }
    }
    
    public enum QuestState {
        COMPLETED, IN_PROGRESS, NOT_STARTED, ANY
    }
    
    public enum ChainLogic {
        ALL {
            @Override
            public boolean evaluate(List<Boolean> results) {
                return results.stream().allMatch(Boolean::booleanValue);
            }
        },
        ANY {
            @Override
            public boolean evaluate(List<Boolean> results) {
                return results.stream().anyMatch(Boolean::booleanValue);
            }
        },
        NONE {
            @Override
            public boolean evaluate(List<Boolean> results) {
                return results.stream().noneMatch(Boolean::booleanValue);
            }
        },
        MAJORITY {
            @Override
            public boolean evaluate(List<Boolean> results) {
                long trueCount = results.stream().filter(Boolean::booleanValue).count();
                return trueCount > results.size() / 2;
            }
        };
        
        public abstract boolean evaluate(List<Boolean> results);
    }
}
```

## Composite Conditions

### Boolean Logic Operators

```java
package com.argonath.framework.condition.composite;

/**
 * AND composite condition - all children must be true.
 */
public class AndCondition implements Condition {
    
    public static final String TYPE = "argonath:and";
    
    private final List<Condition> children;
    
    public AndCondition(List<Condition> children) {
        if (children.isEmpty()) {
            throw new IllegalArgumentException("AND condition requires at least one child");
        }
        this.children = List.copyOf(children);
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        // Short-circuit evaluation: stop on first false
        for (Condition child : children) {
            if (!child.evaluate(context)) {
                return false;
            }
        }
        return true;
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public int getEvaluationCost() {
        // Sum of all children (worst case: all evaluated)
        return children.stream()
            .mapToInt(Condition::getEvaluationCost)
            .sum();
    }
}

/**
 * OR composite condition - at least one child must be true.
 */
public class OrCondition implements Condition {
    
    public static final String TYPE = "argonath:or";
    
    private final List<Condition> children;
    
    public OrCondition(List<Condition> children) {
        if (children.isEmpty()) {
            throw new IllegalArgumentException("OR condition requires at least one child");
        }
        this.children = List.copyOf(children);
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        // Short-circuit evaluation: stop on first true
        for (Condition child : children) {
            if (child.evaluate(context)) {
                return true;
            }
        }
        return false;
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public int getEvaluationCost() {
        // Average cost (expect some short-circuiting)
        return children.stream()
            .mapToInt(Condition::getEvaluationCost)
            .sum() / 2;
    }
}

/**
 * NOT composite condition - inverts child result.
 */
public class NotCondition implements Condition {
    
    public static final String TYPE = "argonath:not";
    
    private final Condition child;
    
    public NotCondition(Condition child) {
        this.child = Objects.requireNonNull(child, "child");
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        return !child.evaluate(context);
    }
    
    @Override
    public String getType() {
        return TYPE;
    }
    
    @Override
    public int getEvaluationCost() {
        return child.getEvaluationCost();
    }
}
```

### Optimization: Cost-Based Ordering

```java
package com.argonath.framework.condition.optimization;

/**
 * Optimized AND condition that evaluates cheapest conditions first.
 */
public class OptimizedAndCondition extends AndCondition {
    
    public OptimizedAndCondition(List<Condition> children) {
        super(optimizeOrder(children));
    }
    
    private static List<Condition> optimizeOrder(List<Condition> children) {
        // Sort by evaluation cost (cheapest first)
        // This maximizes early short-circuit opportunities
        return children.stream()
            .sorted(Comparator.comparingInt(Condition::getEvaluationCost))
            .collect(Collectors.toList());
    }
}

/**
 * Optimized OR condition that evaluates most-likely-true conditions first.
 */
public class OptimizedOrCondition extends OrCondition {
    
    private final ConditionStatistics statistics;
    
    public OptimizedOrCondition(List<Condition> children, ConditionStatistics stats) {
        super(optimizeOrder(children, stats));
        this.statistics = stats;
    }
    
    private static List<Condition> optimizeOrder(List<Condition> children, 
                                                 ConditionStatistics stats) {
        // Sort by success rate (highest first) for better short-circuiting
        return children.stream()
            .sorted(Comparator.comparingDouble(c -> -stats.getSuccessRate(c)))
            .collect(Collectors.toList());
    }
}
```

## Performance Optimization

### Caching Strategies

```java
package com.example.mymod.conditions;

import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;
import java.util.concurrent.TimeUnit;

/**
 * Base class for cacheable conditions.
 */
public abstract class CacheableCondition implements Condition {
    
    private final Cache<String, Boolean> resultCache;
    
    protected CacheableCondition(int cacheSeconds, int maxSize) {
        this.resultCache = CacheBuilder.newBuilder()
            .expireAfterWrite(cacheSeconds, TimeUnit.SECONDS)
            .maximumSize(maxSize)
            .recordStats() // For monitoring
            .build();
    }
    
    @Override
    public final boolean evaluate(ConditionContext context) {
        String cacheKey = buildCacheKey(context);
        
        Boolean cached = resultCache.getIfPresent(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        boolean result = evaluateUncached(context);
        resultCache.put(cacheKey, result);
        return result;
    }
    
    /**
     * Builds a unique cache key for the given context.
     * Must include all relevant context data.
     */
    protected abstract String buildCacheKey(ConditionContext context);
    
    /**
     * Performs the actual evaluation (cache miss).
     */
    protected abstract boolean evaluateUncached(ConditionContext context);
    
    /**
     * Invalidates all cached results.
     */
    public void invalidateCache() {
        resultCache.invalidateAll();
    }
    
    /**
     * Returns cache statistics for monitoring.
     */
    public CacheStats getCacheStats() {
        return resultCache.stats();
    }
}
```

### Lazy Evaluation

```java
package com.argonath.framework.condition.optimization;

/**
 * Lazy-evaluated condition that defers expensive operations.
 */
public class LazyCondition implements Condition {
    
    private final Supplier<Condition> conditionSupplier;
    private volatile Condition delegate;
    
    public LazyCondition(Supplier<Condition> supplier) {
        this.conditionSupplier = supplier;
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        return getDelegate().evaluate(context);
    }
    
    private Condition getDelegate() {
        if (delegate == null) {
            synchronized (this) {
                if (delegate == null) {
                    delegate = conditionSupplier.get();
                }
            }
        }
        return delegate;
    }
    
    @Override
    public String getType() {
        return getDelegate().getType();
    }
    
    @Override
    public int getEvaluationCost() {
        // Return supplier cost if not yet initialized
        return delegate != null ? delegate.getEvaluationCost() : 0;
    }
}
```

## Testing Custom Conditions

### Unit Testing

```java
package com.example.mymod.conditions;

import org.junit.jupiter.api.*;
import org.mockito.Mock;
import org.mockito.MockitoAnnotations;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class AchievementConditionTest {
    
    @Mock
    private ConditionContext context;
    
    @Mock
    private Player player;
    
    @Mock
    private AchievementTracker tracker;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        when(context.getPlayer()).thenReturn(Optional.of(player));
    }
    
    @Test
    @DisplayName("Should return true when achievement is completed")
    void testCompletedAchievement() {
        // Arrange
        when(tracker.hasCompleted("test_achievement")).thenReturn(true);
        AchievementCondition condition = new AchievementCondition("test_achievement", true);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertTrue(result, "Condition should be true for completed achievement");
    }
    
    @Test
    @DisplayName("Should return false when achievement not completed")
    void testNotCompletedAchievement() {
        // Arrange
        when(tracker.hasCompleted("test_achievement")).thenReturn(false);
        AchievementCondition condition = new AchievementCondition("test_achievement", true);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertFalse(result, "Condition should be false for incomplete achievement");
    }
    
    @Test
    @DisplayName("Should handle inverted logic")
    void testInvertedLogic() {
        // Arrange
        when(tracker.hasCompleted("test_achievement")).thenReturn(true);
        AchievementCondition condition = new AchievementCondition("test_achievement", false);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertFalse(result, "Inverted condition should be false when achievement completed");
    }
    
    @Test
    @DisplayName("Should return false when no player in context")
    void testNoPlayerContext() {
        // Arrange
        when(context.getPlayer()).thenReturn(Optional.empty());
        AchievementCondition condition = new AchievementCondition("test_achievement", true);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertFalse(result, "Condition should fail without player context");
    }
    
    @Test
    @DisplayName("Should serialize and deserialize correctly")
    void testSerialization() throws Exception {
        // Arrange
        AchievementCondition original = new AchievementCondition("test_achievement", true);
        
        // Act
        ConfigNode serialized = original.serialize();
        AchievementCondition deserialized = 
            new AchievementConditionDeserializer().deserialize(serialized);
        
        // Assert
        assertEquals(original.getType(), deserialized.getType());
        assertEquals(original.getDescription(), deserialized.getDescription());
    }
}
```

### Integration Testing

```java
package com.example.mymod.conditions.integration;

import com.argonath.framework.test.IntegrationTest;
import com.argonath.framework.test.QuestTestEnvironment;

@IntegrationTest
class ConditionIntegrationTest {
    
    private QuestTestEnvironment env;
    
    @BeforeEach
    void setUp() {
        env = QuestTestEnvironment.create();
    }
    
    @AfterEach
    void tearDown() {
        env.shutdown();
    }
    
    @Test
    @DisplayName("Complex condition chain should work end-to-end")
    void testConditionChain() {
        // Create test player
        TestPlayer player = env.createPlayer("TestPlayer");
        
        // Set up initial state
        env.giveAchievement(player, "first_quest");
        env.setReputation(player, "elves", 50);
        
        // Create complex condition
        Condition condition = new AndCondition(Arrays.asList(
            new AchievementCondition("first_quest", true),
            new ReputationCondition("elves", ComparisonOperator.GREATER_THAN_OR_EQUAL, 40),
            new TimeRangeCondition(8, 18)
        ));
        
        // Set game time to noon
        env.setTime(12, 0);
        
        // Evaluate
        ConditionContext context = env.createContext(player);
        boolean result = condition.evaluate(context);
        
        assertTrue(result, "All conditions should be met");
        
        // Verify cache metrics if applicable
        if (condition instanceof CacheableCondition) {
            CacheableCondition cacheable = (CacheableCondition) condition;
            assertTrue(cacheable.getCacheStats().hitCount() >= 0);
        }
    }
}
```

## Best Practices

### 1. Immutability

Always make conditions immutable. Use final fields and defensive copying:

```java
public class ImmutableCondition implements Condition {
    private final List<String> items;
    
    public ImmutableCondition(List<String> items) {
        this.items = List.copyOf(items); // Defensive copy
    }
}
```

### 2. Thread Safety

Conditions may be evaluated concurrently. Ensure thread-safe implementations:

```java
public class ThreadSafeCondition implements Condition {
    private final AtomicInteger counter = new AtomicInteger();
    
    @Override
    public boolean evaluate(ConditionContext context) {
        counter.incrementAndGet(); // Thread-safe operation
        // ... rest of evaluation
    }
}
```

### 3. Fail Fast

Validate constructor arguments immediately:

```java
public class ValidatedCondition implements Condition {
    public ValidatedCondition(String id, int value) {
        this.id = Objects.requireNonNull(id, "ID cannot be null");
        if (value < 0) {
            throw new IllegalArgumentException("Value must be non-negative");
        }
        this.value = value;
    }
}
```

### 4. Cost Estimation

Provide accurate cost estimates for optimization:

```java
@Override
public int getEvaluationCost() {
    int baseCost = 10;
    int databaseCost = requiresDatabase ? 50 : 0;
    int networkCost = requiresNetwork ? 100 : 0;
    return baseCost + databaseCost + networkCost;
}
```

### 5. Descriptive Errors

Throw meaningful exceptions with context:

```java
@Override
public boolean evaluate(ConditionContext context) {
    try {
        return doEvaluate(context);
    } catch (Exception e) {
        throw new ConditionEvaluationException(
            String.format("Failed to evaluate %s: %s", getType(), getDescription()),
            e
        );
    }
}
```

### 6. Logging

Use structured logging for debugging:

```java
@Override
public boolean evaluate(ConditionContext context) {
    logger.debug("Evaluating condition: type={}, context={}", getType(), context);
    
    boolean result = doEvaluate(context);
    
    logger.debug("Condition result: type={}, result={}, cost={}", 
        getType(), result, getEvaluationCost());
    
    return result;
}
```

## See Also

- [Custom Objectives](custom-objectives.md) - Creating custom objective types
- [Performance Optimization](performance.md) - Advanced performance tuning
- [Testing Strategies](testing.md) - Comprehensive testing guide
- [API Reference](../api/framework-condition.md) - Condition API documentation
