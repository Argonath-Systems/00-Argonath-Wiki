# Testing Strategies

Comprehensive guide to testing quest systems, NPCs, objectives, and custom components in the Argonath Systems framework.

## Table of Contents

- [Overview](#overview)
- [Testing Pyramid](#testing-pyramid)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)
- [End-to-End Testing](#end-to-end-testing)
- [Mocking and Test Doubles](#mocking-and-test-doubles)
- [Test Fixtures and Helpers](#test-fixtures-and-helpers)
- [Automated Testing Workflows](#automated-testing-workflows)
- [Performance Testing](#performance-testing)
- [Best Practices](#best-practices)

## Overview

Comprehensive testing ensures quest systems work correctly under all conditions, from individual conditions to complete quest chains. This guide covers testing strategies specifically for the Argonath framework.

### Testing Goals

- **Correctness**: Verify quest logic works as intended
- **Reliability**: Ensure consistent behavior across environments
- **Performance**: Validate response times and throughput
- **Maintainability**: Catch regressions early
- **Coverage**: Test edge cases and error conditions

## Testing Pyramid

```
                    /\
                   /  \
                  / E2E \          <- Few, slow, expensive
                 /--------\
                /          \
               / Integration \     <- Some, medium speed
              /--------------\
             /                \
            /   Unit Tests     \   <- Many, fast, cheap
           /____________________\
```

### Layer Distribution

- **Unit Tests**: 70% of tests - Fast, isolated, abundant
- **Integration Tests**: 20% - Test component interactions
- **E2E Tests**: 10% - Full system validation

## Unit Testing

### Testing Conditions

```java
package com.example.mymod.conditions;

import org.junit.jupiter.api.*;
import org.mockito.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

/**
 * Unit tests for custom condition implementations.
 */
class PlayerLevelConditionTest {
    
    @Mock
    private ConditionContext context;
    
    @Mock
    private Player player;
    
    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
        when(context.getPlayer()).thenReturn(Optional.of(player));
    }
    
    @Test
    @DisplayName("Should return true when player level meets requirement")
    void testLevelMet() {
        // Arrange
        when(player.getLevel()).thenReturn(10);
        PlayerLevelCondition condition = new PlayerLevelCondition(10, ComparisonOperator.GREATER_THAN_OR_EQUAL);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertTrue(result, "Condition should be true when level requirement is met");
    }
    
    @Test
    @DisplayName("Should return false when player level below requirement")
    void testLevelNotMet() {
        // Arrange
        when(player.getLevel()).thenReturn(5);
        PlayerLevelCondition condition = new PlayerLevelCondition(10, ComparisonOperator.GREATER_THAN_OR_EQUAL);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertFalse(result, "Condition should be false when level requirement not met");
    }
    
    @Test
    @DisplayName("Should handle null player context gracefully")
    void testNullPlayer() {
        // Arrange
        when(context.getPlayer()).thenReturn(Optional.empty());
        PlayerLevelCondition condition = new PlayerLevelCondition(10, ComparisonOperator.GREATER_THAN_OR_EQUAL);
        
        // Act
        boolean result = condition.evaluate(context);
        
        // Assert
        assertFalse(result, "Condition should fail gracefully with no player context");
    }
    
    @Test
    @DisplayName("Should test all comparison operators")
    void testAllOperators() {
        when(player.getLevel()).thenReturn(10);
        
        assertAll("Comparison operators",
            () -> assertTrue(new PlayerLevelCondition(10, ComparisonOperator.EQUAL).evaluate(context)),
            () -> assertTrue(new PlayerLevelCondition(10, ComparisonOperator.GREATER_THAN_OR_EQUAL).evaluate(context)),
            () -> assertTrue(new PlayerLevelCondition(10, ComparisonOperator.LESS_THAN_OR_EQUAL).evaluate(context)),
            () -> assertTrue(new PlayerLevelCondition(9, ComparisonOperator.GREATER_THAN).evaluate(context)),
            () -> assertTrue(new PlayerLevelCondition(11, ComparisonOperator.LESS_THAN).evaluate(context)),
            () -> assertTrue(new PlayerLevelCondition(5, ComparisonOperator.NOT_EQUAL).evaluate(context))
        );
    }
    
    @Test
    @DisplayName("Should serialize and deserialize correctly")
    void testSerialization() throws Exception {
        // Arrange
        PlayerLevelCondition original = new PlayerLevelCondition(10, ComparisonOperator.GREATER_THAN);
        
        // Act
        ConfigNode serialized = original.serialize();
        PlayerLevelCondition deserialized = new PlayerLevelConditionDeserializer().deserialize(serialized);
        
        // Assert
        assertEquals(original.getType(), deserialized.getType());
        assertEquals(original.getDescription(), deserialized.getDescription());
        
        // Verify behavior is identical
        when(player.getLevel()).thenReturn(15);
        assertEquals(original.evaluate(context), deserialized.evaluate(context));
    }
    
    @Nested
    @DisplayName("Edge Cases")
    class EdgeCases {
        
        @Test
        @DisplayName("Should handle maximum integer level")
        void testMaxLevel() {
            when(player.getLevel()).thenReturn(Integer.MAX_VALUE);
            PlayerLevelCondition condition = new PlayerLevelCondition(100, ComparisonOperator.GREATER_THAN);
            
            assertTrue(condition.evaluate(context));
        }
        
        @Test
        @DisplayName("Should handle negative levels")
        void testNegativeLevel() {
            when(player.getLevel()).thenReturn(-1);
            PlayerLevelCondition condition = new PlayerLevelCondition(0, ComparisonOperator.LESS_THAN);
            
            assertTrue(condition.evaluate(context));
        }
        
        @Test
        @DisplayName("Should handle concurrent evaluation")
        void testConcurrentEvaluation() throws InterruptedException {
            PlayerLevelCondition condition = new PlayerLevelCondition(10, ComparisonOperator.EQUAL);
            int threadCount = 100;
            CountDownLatch latch = new CountDownLatch(threadCount);
            AtomicInteger successCount = new AtomicInteger();
            
            // Arrange
            when(player.getLevel()).thenReturn(10);
            
            // Act - Evaluate concurrently from multiple threads
            for (int i = 0; i < threadCount; i++) {
                new Thread(() -> {
                    if (condition.evaluate(context)) {
                        successCount.incrementAndGet();
                    }
                    latch.countDown();
                }).start();
            }
            
            latch.await(5, TimeUnit.SECONDS);
            
            // Assert
            assertEquals(threadCount, successCount.get(), "All concurrent evaluations should succeed");
        }
    }
}
```

### Testing Objectives

```java
package com.example.mymod.objectives;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for objective implementations.
 */
class CollectItemsObjectiveTest {
    
    private CollectItemsObjective objective;
    private UUID testPlayerId;
    
    @BeforeEach
    void setUp() {
        testPlayerId = UUID.randomUUID();
        objective = new CollectItemsObjective(
            "test_collect",
            "minecraft:diamond",
            10,
            false
        );
    }
    
    @Test
    @DisplayName("Should initialize with zero progress")
    void testInitialState() {
        ObjectiveProgress progress = objective.getProgress(testPlayerId);
        
        assertAll("Initial state",
            () -> assertEquals(0, progress.getCurrent(), "Initial progress should be 0"),
            () -> assertEquals(10, progress.getTarget(), "Target should match constructor"),
            () -> assertEquals(0.0, progress.getPercentage(), "Initial percentage should be 0"),
            () -> assertFalse(progress.isComplete(), "Should not be complete initially")
        );
    }
    
    @Test
    @DisplayName("Should update progress correctly")
    void testProgressUpdate() {
        // Create collection event
        ObjectiveEvent event = createItemEvent("minecraft:diamond", 3);
        
        // Update progress
        boolean updated = objective.updateProgress(testPlayerId, event);
        
        // Verify
        assertTrue(updated, "Update should return true");
        assertEquals(3, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @Test
    @DisplayName("Should accumulate multiple updates")
    void testMultipleUpdates() {
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 3));
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 4));
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 2));
        
        assertEquals(9, objective.getProgress(testPlayerId).getCurrent());
        assertFalse(objective.isComplete(testPlayerId));
    }
    
    @Test
    @DisplayName("Should cap progress at target")
    void testProgressCapping() {
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 15));
        
        ObjectiveProgress progress = objective.getProgress(testPlayerId);
        assertEquals(10, progress.getCurrent(), "Progress should be capped at target");
        assertTrue(progress.isComplete());
    }
    
    @Test
    @DisplayName("Should ignore wrong item type")
    void testWrongItemType() {
        boolean updated = objective.updateProgress(
            testPlayerId,
            createItemEvent("minecraft:iron_ingot", 10)
        );
        
        assertFalse(updated, "Should not update for wrong item");
        assertEquals(0, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @Test
    @DisplayName("Should ignore wrong event type")
    void testWrongEventType() {
        ObjectiveEvent wrongEvent = new SimpleObjectiveEvent(
            "block_broken",
            testPlayerId,
            Map.of("block", "minecraft:stone")
        );
        
        boolean updated = objective.updateProgress(testPlayerId, wrongEvent);
        
        assertFalse(updated);
    }
    
    @Test
    @DisplayName("Should track progress independently per player")
    void testMultiplePlayerTracking() {
        UUID player1 = UUID.randomUUID();
        UUID player2 = UUID.randomUUID();
        
        // Update different amounts for each player
        objective.updateProgress(player1, createItemEventFor(player1, "minecraft:diamond", 5));
        objective.updateProgress(player2, createItemEventFor(player2, "minecraft:diamond", 7));
        
        assertEquals(5, objective.getProgress(player1).getCurrent());
        assertEquals(7, objective.getProgress(player2).getCurrent());
    }
    
    @Test
    @DisplayName("Should reset progress correctly")
    void testReset() {
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 7));
        assertEquals(7, objective.getProgress(testPlayerId).getCurrent());
        
        objective.reset(testPlayerId);
        
        assertEquals(0, objective.getProgress(testPlayerId).getCurrent());
    }
    
    @ParameterizedTest
    @CsvSource({
        "1, 10, 0.1",
        "5, 10, 0.5",
        "10, 10, 1.0",
        "0, 10, 0.0",
        "15, 10, 1.0"  // Over-collection capped at 100%
    })
    @DisplayName("Should calculate percentage correctly")
    void testPercentageCalculation(int current, int target, double expected) {
        CollectItemsObjective obj = new CollectItemsObjective("test", "item", target, false);
        obj.updateProgress(testPlayerId, createItemEvent("item", current));
        
        assertEquals(expected, obj.getProgress(testPlayerId).getPercentage(), 0.001);
    }
    
    @Test
    @DisplayName("Should serialize with progress data")
    void testSerializationWithProgress() throws Exception {
        // Add progress
        objective.updateProgress(testPlayerId, createItemEvent("minecraft:diamond", 7));
        
        // Serialize
        ConfigNode serialized = objective.serialize();
        
        // Verify structure
        assertEquals("test_collect", serialized.node("id").getString().get());
        assertEquals(CollectItemsObjective.TYPE, serialized.node("type").getString().get());
        assertEquals("minecraft:diamond", serialized.node("item").getString().get());
        assertEquals(10, serialized.node("quantity").getInt().get());
        
        // Verify progress saved
        assertTrue(serialized.node("progress", testPlayerId.toString()).isPresent());
        assertEquals(7, serialized.node("progress", testPlayerId.toString()).getInt().get());
    }
    
    @Nested
    @DisplayName("Consume mode tests")
    class ConsumeModeTests {
        
        @Test
        @DisplayName("Should indicate consume mode in description")
        void testConsumeDescription() {
            CollectItemsObjective consumeObj = new CollectItemsObjective(
                "test", "minecraft:diamond", 10, true
            );
            
            assertTrue(consumeObj.getDescription().contains("Consume"));
        }
    }
    
    // Helper methods
    private ObjectiveEvent createItemEvent(String itemId, int quantity) {
        return createItemEventFor(testPlayerId, itemId, quantity);
    }
    
    private ObjectiveEvent createItemEventFor(UUID playerId, String itemId, int quantity) {
        return new SimpleObjectiveEvent(
            "item_collected",
            playerId,
            Map.of("item_id", itemId, "quantity", quantity)
        );
    }
}
```

### Testing Quest Logic

```java
package com.example.mymod.quests;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Unit tests for quest implementations.
 */
class QuestTest {
    
    private Quest quest;
    private UUID testPlayerId;
    
    @BeforeEach
    void setUp() {
        testPlayerId = UUID.randomUUID();
        
        // Build a simple quest
        quest = new QuestBuilder()
            .id("test_quest")
            .name("Test Quest")
            .description("A test quest")
            .addObjective(new CollectItemsObjective("obj1", "minecraft:diamond", 5, false))
            .addObjective(new PlayerLevelObjective("obj2", 10))
            .addReward(new XpReward(100))
            .build();
    }
    
    @Test
    @DisplayName("Quest should not be complete initially")
    void testInitialState() {
        QuestProgress progress = quest.getProgress(testPlayerId);
        
        assertFalse(progress.isComplete());
        assertEquals(0, progress.getCompletedObjectives());
        assertEquals(2, progress.getTotalObjectives());
    }
    
    @Test
    @DisplayName("Should track objective completion")
    void testObjectiveCompletion() {
        // Complete first objective
        ObjectiveEvent event = new SimpleObjectiveEvent(
            "item_collected",
            testPlayerId,
            Map.of("item_id", "minecraft:diamond", "quantity", 5)
        );
        
        quest.notifyEvent(testPlayerId, event);
        
        QuestProgress progress = quest.getProgress(testPlayerId);
        assertEquals(1, progress.getCompletedObjectives());
        assertFalse(progress.isComplete());
    }
    
    @Test
    @DisplayName("Should complete when all objectives done")
    void testQuestCompletion() {
        // Complete both objectives
        quest.notifyEvent(testPlayerId, createItemEvent("minecraft:diamond", 5));
        quest.setPlayerLevel(testPlayerId, 10); // Mock level up
        
        assertTrue(quest.isComplete(testPlayerId));
    }
    
    @Test
    @DisplayName("Should grant rewards on completion")
    void testRewardGrant() {
        // Mock reward system
        RewardTracker tracker = mock(RewardTracker.class);
        quest.setRewardTracker(tracker);
        
        // Complete quest
        completeAllObjectives();
        quest.claimRewards(testPlayerId);
        
        // Verify reward granted
        verify(tracker).grantReward(eq(testPlayerId), any(XpReward.class));
    }
}
```

## Integration Testing

### Test Environment Setup

```java
package com.argonath.framework.test;

/**
 * Integration test environment for quest systems.
 */
public class QuestTestEnvironment {
    
    private final TestServer server;
    private final TestDatabase database;
    private final QuestManager questManager;
    private final Map<UUID, TestPlayer> players;
    
    private QuestTestEnvironment() {
        this.server = new TestServer();
        this.database = new TestDatabase();
        this.questManager = new QuestManager(database.getStorageAdapter());
        this.players = new HashMap<>();
    }
    
    /**
     * Creates a new test environment.
     */
    public static QuestTestEnvironment create() {
        QuestTestEnvironment env = new QuestTestEnvironment();
        env.initialize();
        return env;
    }
    
    /**
     * Initializes the test environment.
     */
    private void initialize() {
        server.start();
        database.initialize();
        questManager.initialize();
    }
    
    /**
     * Creates a test player.
     */
    public TestPlayer createPlayer(String name) {
        TestPlayer player = new TestPlayer(UUID.randomUUID(), name);
        players.put(player.getUUID(), player);
        server.addPlayer(player);
        return player;
    }
    
    /**
     * Starts a quest for a player.
     */
    public void startQuest(TestPlayer player, Quest quest) {
        questManager.startQuest(player.getUUID(), quest);
    }
    
    /**
     * Simulates giving items to a player.
     */
    public void giveItem(TestPlayer player, String itemId, int quantity) {
        player.getInventory().addItem(itemId, quantity);
        
        // Trigger collection event
        ObjectiveEvent event = new SimpleObjectiveEvent(
            "item_collected",
            player.getUUID(),
            Map.of("item_id", itemId, "quantity", quantity)
        );
        
        questManager.notifyObjectiveEvent(player.getUUID(), event);
    }
    
    /**
     * Simulates player leveling up.
     */
    public void setLevel(TestPlayer player, int level) {
        int oldLevel = player.getLevel();
        player.setLevel(level);
        
        // Trigger level change event
        if (level > oldLevel) {
            ObjectiveEvent event = new SimpleObjectiveEvent(
                "level_gained",
                player.getUUID(),
                Map.of("old_level", oldLevel, "new_level", level)
            );
            
            questManager.notifyObjectiveEvent(player.getUUID(), event);
        }
    }
    
    /**
     * Sets game time.
     */
    public void setTime(int hour, int minute) {
        server.setTime(hour * 60 + minute);
    }
    
    /**
     * Gets objective progress.
     */
    public ObjectiveProgress getObjectiveProgress(TestPlayer player, Quest quest, int objectiveIndex) {
        return quest.getObjectives().get(objectiveIndex).getProgress(player.getUUID());
    }
    
    /**
     * Checks if objective is complete.
     */
    public boolean isObjectiveComplete(TestPlayer player, Quest quest, int objectiveIndex) {
        return getObjectiveProgress(player, quest, objectiveIndex).isComplete();
    }
    
    /**
     * Checks if quest is complete.
     */
    public boolean isQuestComplete(TestPlayer player, Quest quest) {
        return quest.isComplete(player.getUUID());
    }
    
    /**
     * Creates a condition context for testing.
     */
    public ConditionContext createContext(TestPlayer player) {
        return new TestConditionContext(player, server.getWorld(), server.getTime());
    }
    
    /**
     * Shuts down the environment.
     */
    public void shutdown() {
        questManager.shutdown();
        database.shutdown();
        server.stop();
    }
}
```

### Integration Test Examples

```java
package com.example.mymod.integration;

import com.argonath.framework.test.*;
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

/**
 * Integration tests for quest system.
 */
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class QuestIntegrationTest {
    
    private QuestTestEnvironment env;
    
    @BeforeAll
    void setUpEnvironment() {
        env = QuestTestEnvironment.create();
    }
    
    @AfterAll
    void tearDownEnvironment() {
        env.shutdown();
    }
    
    @Test
    @DisplayName("Complete quest chain integration test")
    void testQuestChain() {
        // Create player
        TestPlayer player = env.createPlayer("TestPlayer");
        
        // Create quest chain: Quest1 -> Quest2 -> Quest3
        Quest quest1 = new QuestBuilder()
            .id("chain_1")
            .name("First Steps")
            .objective(new CollectItemsBuilder()
                .item("minecraft:wood")
                .quantity(10)
                .build())
            .reward(new XpReward(50))
            .build();
            
        Quest quest2 = new QuestBuilder()
            .id("chain_2")
            .name("Getting Tools")
            .requirement(new QuestCompleteCondition("chain_1"))
            .objective(new CraftingBuilder()
                .item("minecraft:wooden_pickaxe")
                .build())
            .reward(new XpReward(100))
            .build();
            
        Quest quest3 = new QuestBuilder()
            .id("chain_3")
            .name("Mining Time")
            .requirement(new QuestCompleteCondition("chain_2"))
            .objective(new MiningBuilder()
                .block("minecraft:stone")
                .quantity(50)
                .build())
            .reward(new ItemReward("minecraft:diamond", 1))
            .build();
        
        // Start first quest
        env.startQuest(player, quest1);
        
        // Complete first quest
        env.giveItem(player, "minecraft:wood", 10);
        assertTrue(env.isQuestComplete(player, quest1));
        
        // Second quest should now be available
        env.startQuest(player, quest2);
        
        // Complete second quest
        env.craftItem(player, "minecraft:wooden_pickaxe");
        assertTrue(env.isQuestComplete(player, quest2));
        
        // Third quest available
        env.startQuest(player, quest3);
        
        // Complete third quest
        env.mineBlocks(player, "minecraft:stone", 50);
        assertTrue(env.isQuestComplete(player, quest3));
        
        // Verify final rewards
        assertTrue(player.getInventory().hasItem("minecraft:diamond", 1));
    }
    
    @Test
    @DisplayName("Parallel quest execution")
    void testParallelQuests() {
        TestPlayer player = env.createPlayer("ParallelTester");
        
        // Create multiple independent quests
        Quest quest1 = createCollectQuest("parallel_1", "minecraft:wheat", 20);
        Quest quest2 = createCollectQuest("parallel_2", "minecraft:carrot", 15);
        Quest quest3 = createMiningQuest("parallel_3", "minecraft:coal_ore", 10);
        
        // Start all quests
        env.startQuest(player, quest1);
        env.startQuest(player, quest2);
        env.startQuest(player, quest3);
        
        // Make progress on all quests
        env.giveItem(player, "minecraft:wheat", 10);
        env.giveItem(player, "minecraft:carrot", 5);
        env.mineBlocks(player, "minecraft:coal_ore", 3);
        
        // Verify progress
        assertEquals(0.5, env.getObjectiveProgress(player, quest1, 0).getPercentage(), 0.01);
        assertEquals(0.33, env.getObjectiveProgress(player, quest2, 0).getPercentage(), 0.01);
        assertEquals(0.3, env.getObjectiveProgress(player, quest3, 0).getPercentage(), 0.01);
        
        // Complete all quests
        env.giveItem(player, "minecraft:wheat", 10);
        env.giveItem(player, "minecraft:carrot", 10);
        env.mineBlocks(player, "minecraft:coal_ore", 7);
        
        assertTrue(env.isQuestComplete(player, quest1));
        assertTrue(env.isQuestComplete(player, quest2));
        assertTrue(env.isQuestComplete(player, quest3));
    }
    
    @Test
    @DisplayName("Quest persistence across sessions")
    void testQuestPersistence() {
        TestPlayer player = env.createPlayer("PersistenceTester");
        
        Quest quest = new QuestBuilder()
            .id("persist_test")
            .objective(new CollectItemsBuilder()
                .item("minecraft:diamond")
                .quantity(10)
                .build())
            .build();
        
        // Start quest and make partial progress
        env.startQuest(player, quest);
        env.giveItem(player, "minecraft:diamond", 6);
        
        // Simulate server restart
        env.saveAndRestart();
        
        // Verify progress restored
        Quest loadedQuest = env.getQuestManager().getQuest("persist_test");
        ObjectiveProgress progress = loadedQuest.getObjectives().get(0).getProgress(player.getUUID());
        
        assertEquals(6, progress.getCurrent());
        
        // Complete quest after restart
        env.giveItem(player, "minecraft:diamond", 4);
        assertTrue(env.isQuestComplete(player, loadedQuest));
    }
    
    @Test
    @DisplayName("Concurrent player quest execution")
    void testConcurrentPlayers() throws InterruptedException {
        int playerCount = 50;
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch completeLatch = new CountDownLatch(playerCount);
        
        Quest sharedQuest = createCollectQuest("concurrent_test", "minecraft:stone", 10);
        
        // Create multiple players
        List<TestPlayer> players = new ArrayList<>();
        for (int i = 0; i < playerCount; i++) {
            TestPlayer player = env.createPlayer("Player" + i);
            players.add(player);
            env.startQuest(player, sharedQuest);
        }
        
        // Have all players complete quest concurrently
        for (TestPlayer player : players) {
            new Thread(() -> {
                try {
                    startLatch.await(); // Wait for signal
                    env.giveItem(player, "minecraft:stone", 10);
                    completeLatch.countDown();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }).start();
        }
        
        // Signal all threads to start
        startLatch.countDown();
        
        // Wait for all to complete
        assertTrue(completeLatch.await(10, TimeUnit.SECONDS));
        
        // Verify all players completed
        for (TestPlayer player : players) {
            assertTrue(env.isQuestComplete(player, sharedQuest),
                "Player " + player.getName() + " should have completed quest");
        }
    }
    
    // Helper methods
    private Quest createCollectQuest(String id, String itemId, int quantity) {
        return new QuestBuilder()
            .id(id)
            .objective(new CollectItemsBuilder()
                .item(itemId)
                .quantity(quantity)
                .build())
            .build();
    }
    
    private Quest createMiningQuest(String id, String blockType, int quantity) {
        return new QuestBuilder()
            .id(id)
            .objective(new MiningBuilder()
                .block(blockType)
                .quantity(quantity)
                .build())
            .build();
    }
}
```

## End-to-End Testing

### E2E Test Framework

```java
package com.argonath.framework.test.e2e;

/**
 * End-to-end test framework using real server instance.
 */
public class E2ETestFramework {
    
    private final ServerInstance server;
    private final TestClient[] clients;
    private final DatabaseInstance database;
    
    public E2ETestFramework(int clientCount) {
        this.database = DatabaseInstance.createTestInstance();
        this.server = ServerInstance.create(database);
        this.clients = new TestClient[clientCount];
        
        for (int i = 0; i < clientCount; i++) {
            clients[i] = new TestClient("TestClient" + i);
        }
    }
    
    /**
     * Starts the server and connects clients.
     */
    public void start() throws Exception {
        database.start();
        server.start();
        
        // Connect all clients
        for (TestClient client : clients) {
            client.connect(server.getAddress());
        }
        
        // Wait for all clients to be ready
        waitForClientsReady(Duration.ofSeconds(30));
    }
    
    /**
     * Runs an E2E test scenario.
     */
    public void runScenario(E2EScenario scenario) {
        scenario.execute(this);
    }
    
    /**
     * Gets a client by index.
     */
    public TestClient getClient(int index) {
        return clients[index];
    }
    
    /**
     * Broadcasts a quest to all clients.
     */
    public void broadcastQuest(Quest quest) {
        for (TestClient client : clients) {
            client.receiveQuest(quest);
        }
    }
    
    /**
     * Shuts down the framework.
     */
    public void shutdown() throws Exception {
        for (TestClient client : clients) {
            client.disconnect();
        }
        
        server.stop();
        database.stop();
    }
    
    private void waitForClientsReady(Duration timeout) throws TimeoutException {
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        
        while (System.currentTimeMillis() < deadline) {
            boolean allReady = Arrays.stream(clients)
                .allMatch(TestClient::isReady);
                
            if (allReady) {
                return;
            }
            
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }
        
        throw new TimeoutException("Clients not ready within timeout");
    }
}
```

### E2E Test Scenarios

```java
package com.example.mymod.e2e;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

/**
 * End-to-end test scenarios.
 */
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class QuestE2ETest {
    
    private static E2ETestFramework framework;
    
    @BeforeAll
    static void setUpFramework() throws Exception {
        framework = new E2ETestFramework(5); // 5 test clients
        framework.start();
    }
    
    @AfterAll
    static void tearDownFramework() throws Exception {
        framework.shutdown();
    }
    
    @Test
    @Order(1)
    @DisplayName("E2E: Player receives and accepts quest")
    void testQuestAcceptance() {
        TestClient client = framework.getClient(0);
        
        // Server sends quest to player
        Quest quest = createSimpleQuest();
        client.receiveQuest(quest);
        
        // Verify client received quest
        assertTrue(client.hasQuest(quest.getId()));
        
        // Player accepts quest
        client.acceptQuest(quest.getId());
        
        // Verify quest is active
        assertTrue(client.isQuestActive(quest.getId()));
    }
    
    @Test
    @Order(2)
    @DisplayName("E2E: Player completes quest and receives rewards")
    void testQuestCompletion() throws InterruptedException {
        TestClient client = framework.getClient(0);
        
        Quest quest = new QuestBuilder()
            .id("e2e_complete")
            .objective(new CollectItemsBuilder()
                .item("minecraft:stone")
                .quantity(10)
                .build())
            .reward(new XpReward(100))
            .build();
        
        // Send and accept quest
        client.receiveQuest(quest);
        client.acceptQuest(quest.getId());
        
        // Simulate collecting items
        for (int i = 0; i < 10; i++) {
            client.collectItem("minecraft:stone", 1);
            Thread.sleep(50); // Simulate time between collections
        }
        
        // Wait for server to process
        Thread.sleep(200);
        
        // Verify quest completed
        assertTrue(client.isQuestCompleted(quest.getId()));
        
        // Claim rewards
        client.claimRewards(quest.getId());
        
        // Verify rewards received
        assertTrue(client.hasReceivedXp(100));
    }
    
    @Test
    @Order(3)
    @DisplayName("E2E: Multiple players complete same quest")
    void testMultiPlayerQuest() throws InterruptedException {
        Quest sharedQuest = new QuestBuilder()
            .id("e2e_multiplayer")
            .objective(new MiningBuilder()
                .block("minecraft:coal_ore")
                .quantity(5)
                .build())
            .build();
        
        // Broadcast quest to all clients
        framework.broadcastQuest(sharedQuest);
        
        // All clients accept
        for (int i = 0; i < 5; i++) {
            framework.getClient(i).acceptQuest(sharedQuest.getId());
        }
        
        // Clients complete at different rates
        CountDownLatch completionLatch = new CountDownLatch(5);
        
        for (int i = 0; i < 5; i++) {
            final int clientIndex = i;
            new Thread(() -> {
                TestClient client = framework.getClient(clientIndex);
                for (int j = 0; j < 5; j++) {
                    client.mineBlock("minecraft:coal_ore");
                    try {
                        Thread.sleep(100 * (clientIndex + 1)); // Different speeds
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
                completionLatch.countDown();
            }).start();
        }
        
        // Wait for all clients to complete
        assertTrue(completionLatch.await(10, TimeUnit.SECONDS));
        
        // Verify all completed
        Thread.sleep(500); // Allow server processing
        for (int i = 0; i < 5; i++) {
            assertTrue(framework.getClient(i).isQuestCompleted(sharedQuest.getId()));
        }
    }
    
    @Test
    @Order(4)
    @DisplayName("E2E: Quest progression survives server restart")
    void testServerRestart() throws Exception {
        TestClient client = framework.getClient(0);
        
        Quest quest = new QuestBuilder()
            .id("e2e_persistence")
            .objective(new CollectItemsBuilder()
                .item("minecraft:diamond")
                .quantity(20)
                .build())
            .build();
        
        // Start quest and make partial progress
        client.receiveQuest(quest);
        client.acceptQuest(quest.getId());
        client.collectItem("minecraft:diamond", 12);
        
        Thread.sleep(200); // Ensure saved
        
        // Restart server
        framework.restartServer();
        
        // Reconnect client
        client.reconnect();
        
        Thread.sleep(500); // Allow reconnection
        
        // Verify progress restored
        assertEquals(12, client.getObjectiveProgress(quest.getId(), 0));
        
        // Continue and complete
        client.collectItem("minecraft:diamond", 8);
        Thread.sleep(200);
        
        assertTrue(client.isQuestCompleted(quest.getId()));
    }
    
    private Quest createSimpleQuest() {
        return new QuestBuilder()
            .id("e2e_simple")
            .name("Simple Quest")
            .description("A simple E2E test quest")
            .objective(new CollectItemsBuilder()
                .item("minecraft:dirt")
                .quantity(1)
                .build())
            .build();
    }
}
```

## Mocking and Test Doubles

### Platform Mocks

```java
package com.argonath.framework.test.mocks;

import org.mockito.Mockito;
import static org.mockito.Mockito.*;

/**
 * Factory for creating platform mocks.
 */
public class PlatformMocks {
    
    /**
     * Creates a mock player with default behavior.
     */
    public static Player createMockPlayer(UUID id, String name) {
        Player player = mock(Player.class);
        
        when(player.getUUID()).thenReturn(id);
        when(player.getName()).thenReturn(name);
        when(player.getLevel()).thenReturn(1);
        when(player.getInventory()).thenReturn(createMockInventory());
        
        return player;
    }
    
    /**
     * Creates a mock inventory.
     */
    public static Inventory createMockInventory() {
        Inventory inventory = mock(Inventory.class);
        
        Map<String, Integer> items = new HashMap<>();
        
        when(inventory.hasItem(anyString(), anyInt())).thenAnswer(inv -> {
            String itemId = inv.getArgument(0);
            int required = inv.getArgument(1);
            return items.getOrDefault(itemId, 0) >= required;
        });
        
        when(inventory.addItem(anyString(), anyInt())).thenAnswer(inv -> {
            String itemId = inv.getArgument(0);
            int quantity = inv.getArgument(1);
            items.merge(itemId, quantity, Integer::sum);
            return null;
        });
        
        return inventory;
    }
    
    /**
     * Creates a mock condition context.
     */
    public static ConditionContext createMockContext(Player player) {
        ConditionContext context = mock(ConditionContext.class);
        
        when(context.getPlayer()).thenReturn(Optional.ofNullable(player));
        when(context.getWorld()).thenReturn(createMockWorld());
        when(context.getServerTime()).thenReturn(System.currentTimeMillis());
        
        return context;
    }
    
    /**
     * Creates a mock world.
     */
    public static World createMockWorld() {
        World world = mock(World.class);
        
        when(world.getName()).thenReturn("test_world");
        when(world.getTime()).thenReturn(0L);
        when(world.isDay()).thenReturn(true);
        
        return world;
    }
    
    /**
     * Creates a mock storage adapter.
     */
    public static StorageAdapter createMockStorage() {
        StorageAdapter storage = mock(StorageAdapter.class);
        
        // In-memory storage simulation
        Map<String, Quest> quests = new ConcurrentHashMap<>();
        Map<String, Map<UUID, Double>> progress = new ConcurrentHashMap<>();
        
        when(storage.saveQuest(any(Quest.class))).thenAnswer(inv -> {
            Quest quest = inv.getArgument(0);
            quests.put(quest.getId(), quest);
            return CompletableFuture.completedFuture(null);
        });
        
        when(storage.loadQuest(anyString())).thenAnswer(inv -> {
            String questId = inv.getArgument(0);
            return Optional.ofNullable(quests.get(questId));
        });
        
        return storage;
    }
}
```

### Test Doubles

```java
package com.argonath.framework.test.doubles;

/**
 * Test double for condition that allows controlling evaluation result.
 */
public class StubCondition implements Condition {
    
    private boolean result;
    private int evaluationCount = 0;
    
    public StubCondition(boolean result) {
        this.result = result;
    }
    
    @Override
    public boolean evaluate(ConditionContext context) {
        evaluationCount++;
        return result;
    }
    
    public void setResult(boolean result) {
        this.result = result;
    }
    
    public int getEvaluationCount() {
        return evaluationCount;
    }
    
    @Override
    public String getType() {
        return "test:stub";
    }
    
    // ... other interface methods
}

/**
 * Spy objective that records all method calls.
 */
public class SpyObjective implements Objective {
    
    private final Objective delegate;
    private final List<MethodCall> calls = new ArrayList<>();
    
    public SpyObjective(Objective delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public boolean updateProgress(UUID playerId, ObjectiveEvent event) {
        calls.add(new MethodCall("updateProgress", playerId, event));
        return delegate.updateProgress(playerId, event);
    }
    
    @Override
    public ObjectiveProgress getProgress(UUID playerId) {
        calls.add(new MethodCall("getProgress", playerId));
        return delegate.getProgress(playerId);
    }
    
    public List<MethodCall> getCalls() {
        return Collections.unmodifiableList(calls);
    }
    
    public int getCallCount(String methodName) {
        return (int) calls.stream()
            .filter(call -> call.methodName.equals(methodName))
            .count();
    }
    
    // ... delegate other methods
    
    public static class MethodCall {
        final String methodName;
        final Object[] args;
        final long timestamp;
        
        MethodCall(String methodName, Object... args) {
            this.methodName = methodName;
            this.args = args;
            this.timestamp = System.currentTimeMillis();
        }
    }
}
```

## Test Fixtures and Helpers

### Quest Test Fixtures

```java
package com.argonath.framework.test.fixtures;

/**
 * Provides pre-configured quest fixtures for testing.
 */
public class QuestFixtures {
    
    /**
     * Creates a simple single-objective quest.
     */
    public static Quest createSimpleQuest() {
        return new QuestBuilder()
            .id("simple_quest")
            .name("Simple Quest")
            .objective(new CollectItemsBuilder()
                .item("minecraft:stone")
                .quantity(10)
                .build())
            .build();
    }
    
    /**
     * Creates a complex multi-objective quest.
     */
    public static Quest createComplexQuest() {
        return new QuestBuilder()
            .id("complex_quest")
            .name("Complex Quest")
            .objective(new CollectItemsBuilder()
                .item("minecraft:wood")
                .quantity(20)
                .build())
            .objective(new CraftingBuilder()
                .item("minecraft:wooden_pickaxe")
                .build())
            .objective(new MiningBuilder()
                .block("minecraft:stone")
                .quantity(50)
                .build())
            .requirement(new PlayerLevelCondition(5, ComparisonOperator.GREATER_THAN_OR_EQUAL))
            .reward(new XpReward(500))
            .reward(new ItemReward("minecraft:diamond", 3))
            .build();
    }
    
    /**
     * Creates a quest chain.
     */
    public static List<Quest> createQuestChain(int length) {
        List<Quest> chain = new ArrayList<>();
        
        for (int i = 0; i < length; i++) {
            QuestBuilder builder = new QuestBuilder()
                .id("chain_" + i)
                .name("Quest " + (i + 1))
                .objective(new CollectItemsBuilder()
                    .item("minecraft:item_" + i)
                    .quantity(10)
                    .build());
            
            // Add requirement for previous quest
            if (i > 0) {
                builder.requirement(new QuestCompleteCondition("chain_" + (i - 1)));
            }
            
            chain.add(builder.build());
        }
        
        return chain;
    }
}
```

### Assertion Helpers

```java
package com.argonath.framework.test.assertions;

/**
 * Custom assertions for quest testing.
 */
public class QuestAssertions {
    
    /**
     * Asserts that a quest is complete.
     */
    public static void assertQuestComplete(Quest quest, UUID playerId) {
        assertTrue(quest.isComplete(playerId),
            String.format("Quest '%s' should be complete for player %s", 
                quest.getId(), playerId));
    }
    
    /**
     * Asserts objective progress.
     */
    public static void assertProgress(Objective objective, UUID playerId, 
                                      double expected, double delta) {
        double actual = objective.getProgress(playerId).getCurrent();
        assertEquals(expected, actual, delta,
            String.format("Objective '%s' progress should be %.2f, was %.2f",
                objective.getId(), expected, actual));
    }
    
    /**
     * Asserts that a condition evaluates to expected result.
     */
    public static void assertCondition(Condition condition, ConditionContext context, 
                                       boolean expected) {
        boolean actual = condition.evaluate(context);
        assertEquals(expected, actual,
            String.format("Condition '%s' should evaluate to %b, was %b",
                condition.getType(), expected, actual));
    }
    
    /**
     * Asserts quest progress percentage.
     */
    public static void assertQuestProgress(Quest quest, UUID playerId, 
                                          double expectedPercentage, double delta) {
        QuestProgress progress = quest.getProgress(playerId);
        double actual = (double) progress.getCompletedObjectives() / progress.getTotalObjectives();
        assertEquals(expectedPercentage, actual, delta,
            String.format("Quest '%s' should be %.0f%% complete",
                quest.getId(), expectedPercentage * 100));
    }
}
```

## Automated Testing Workflows

### CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Test Quest System

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_DB: quest_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}
      
      - name: Run unit tests
        run: ./gradlew test
      
      - name: Run integration tests
        run: ./gradlew integrationTest
        env:
          DB_URL: jdbc:postgresql://localhost:5432/quest_test
          DB_USER: test
          DB_PASSWORD: test
      
      - name: Generate coverage report
        run: ./gradlew jacocoTestReport
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./build/reports/jacoco/test/jacocoTestReport.xml
      
      - name: Archive test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: build/test-results/
```

## Performance Testing

### Load Testing

```java
package com.argonath.framework.test.performance;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

/**
 * JMH benchmarks for quest system performance.
 */
@State(Scope.Benchmark)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 3, time = 5)
@Measurement(iterations = 5, time = 10)
@Fork(1)
public class QuestBenchmarks {
    
    private Quest quest;
    private UUID playerId;
    private ConditionContext context;
    
    @Setup
    public void setUp() {
        playerId = UUID.randomUUID();
        quest = QuestFixtures.createComplexQuest();
        context = PlatformMocks.createMockContext(
            PlatformMocks.createMockPlayer(playerId, "TestPlayer")
        );
    }
    
    @Benchmark
    public boolean benchmarkQuestEvaluation() {
        return quest.isComplete(playerId);
    }
    
    @Benchmark
    public void benchmarkObjectiveUpdate() {
        ObjectiveEvent event = new SimpleObjectiveEvent(
            "item_collected",
            playerId,
            Map.of("item_id", "minecraft:stone", "quantity", 1)
        );
        quest.notifyEvent(playerId, event);
    }
    
    @Benchmark
    public boolean benchmarkConditionEvaluation() {
        return quest.getRequirements().get(0).evaluate(context);
    }
}
```

## Best Practices

### 1. Test Naming

Use descriptive test names that explain what is being tested:

```java
@Test
@DisplayName("Should complete quest when all objectives are met")
void testQuestCompletionWithAllObjectivesMet() {
    // ...
}
```

### 2. AAA Pattern

Structure tests with Arrange-Act-Assert:

```java
@Test
void testExample() {
    // Arrange: Set up test data
    Quest quest = QuestFixtures.createSimpleQuest();
    UUID playerId = UUID.randomUUID();
    
    // Act: Execute the behavior
    quest.startQuest(playerId);
    
    // Assert: Verify results
    assertTrue(quest.isActive(playerId));
}
```

### 3. Test Isolation

Each test should be independent:

```java
@BeforeEach
void setUp() {
    // Fresh state for each test
    quest = new QuestBuilder().build();
    playerId = UUID.randomUUID();
}
```

### 4. Use Parameterized Tests

Test multiple cases efficiently:

```java
@ParameterizedTest
@ValueSource(ints = {1, 5, 10, 100})
void testDifferentQuantities(int quantity) {
    // Test with different values
}
```

### 5. Verify Error Handling

```java
@Test
void testInvalidInput() {
    assertThrows(IllegalArgumentException.class, () -> {
        new CollectItemsObjective("id", null, 10, false);
    });
}
```

## See Also

- [Custom Conditions](custom-conditions.md) - Creating custom condition types
- [Custom Objectives](custom-objectives.md) - Creating custom objective types
- [Performance Optimization](performance.md) - Advanced performance tuning
- [Development Guide](../guides/development.md) - Development workflows
