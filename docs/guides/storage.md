# Storage Guide

A comprehensive guide to data persistence and storage in the Argonath Framework.

## Table of Contents

1. [Introduction](#introduction)
2. [Storage Architecture](#storage-architecture)
3. [Storage Types](#storage-types)
4. [Serialization and Deserialization](#serialization-and-deserialization)
5. [Database Integration](#database-integration)
6. [Player Data Storage](#player-data-storage)
7. [World Data Storage](#world-data-storage)
8. [Quest Data Persistence](#quest-data-persistence)
9. [Caching Strategies](#caching-strategies)
10. [Migration Strategies](#migration-strategies)
11. [Performance Optimization](#performance-optimization)
12. [Backup and Recovery](#backup-and-recovery)
13. [Best Practices](#best-practices)
14. [Troubleshooting](#troubleshooting)

---

## Introduction

The Argonath Framework Storage System provides a robust, scalable solution for persisting game data. It supports multiple storage backends, automatic serialization, caching, and migration strategies.

### Prerequisites

- Understanding of [Core Concepts](../architecture/concepts.md)
- Basic database knowledge (SQL/NoSQL)
- Java serialization concepts
- Familiarity with [Configuration Guide](configuration.md)

### What You'll Learn

- Storage system architecture
- Different storage types and when to use them
- Serialization and deserialization patterns
- Database integration
- Performance optimization techniques
- Data migration strategies

---

## Storage Architecture

### Core Components

```
StorageManager
    ├── StorageProvider (SQLite, MySQL, MongoDB, File)
    ├── DataSerializer (JSON, Binary, NBT)
    ├── CacheManager
    ├── MigrationManager
    └── BackupManager

Storage Hierarchy:
    ├── Global Storage (Server-wide data)
    ├── World Storage (Per-world data)
    └── Player Storage (Per-player data)
```

### Storage Interface

```java
public interface StorageProvider {
    // Basic operations
    <T> void save(String key, T data);
    <T> T load(String key, Class<T> type);
    void delete(String key);
    boolean exists(String key);
    
    // Batch operations
    <T> void saveAll(Map<String, T> data);
    <T> Map<String, T> loadAll(Class<T> type);
    
    // Query operations
    <T> List<T> query(Query query, Class<T> type);
    <T> T queryFirst(Query query, Class<T> type);
    
    // Lifecycle
    void initialize();
    void shutdown();
    void flush();
}
```

---

## Storage Types

### 1. File-Based Storage

Simple file-based storage for small to medium datasets.

```java
public class FileStorage implements StorageProvider {
    private final Path dataDirectory;
    private final DataSerializer serializer;
    
    public FileStorage(Path dataDirectory, DataSerializer serializer) {
        this.dataDirectory = dataDirectory;
        this.serializer = serializer;
    }
    
    @Override
    public <T> void save(String key, T data) {
        try {
            Path filePath = dataDirectory.resolve(key + ".json");
            Files.createDirectories(filePath.getParent());
            
            String json = serializer.serialize(data);
            Files.writeString(filePath, json, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new StorageException("Failed to save data: " + key, e);
        }
    }
    
    @Override
    public <T> T load(String key, Class<T> type) {
        try {
            Path filePath = dataDirectory.resolve(key + ".json");
            
            if (!Files.exists(filePath)) {
                return null;
            }
            
            String json = Files.readString(filePath, StandardCharsets.UTF_8);
            return serializer.deserialize(json, type);
        } catch (IOException e) {
            throw new StorageException("Failed to load data: " + key, e);
        }
    }
    
    @Override
    public void delete(String key) {
        try {
            Path filePath = dataDirectory.resolve(key + ".json");
            Files.deleteIfExists(filePath);
        } catch (IOException e) {
            throw new StorageException("Failed to delete data: " + key, e);
        }
    }
    
    @Override
    public boolean exists(String key) {
        Path filePath = dataDirectory.resolve(key + ".json");
        return Files.exists(filePath);
    }
}
```

### 2. SQLite Storage

Lightweight embedded database for structured data.

```java
public class SQLiteStorage implements StorageProvider {
    private final String databasePath;
    private Connection connection;
    
    public SQLiteStorage(String databasePath) {
        this.databasePath = databasePath;
    }
    
    @Override
    public void initialize() {
        try {
            Class.forName("org.sqlite.JDBC");
            connection = DriverManager.getConnection("jdbc:sqlite:" + databasePath);
            createTables();
        } catch (Exception e) {
            throw new StorageException("Failed to initialize SQLite", e);
        }
    }
    
    private void createTables() throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            // Player data table
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS player_data (
                    uuid TEXT PRIMARY KEY,
                    data TEXT NOT NULL,
                    last_updated INTEGER NOT NULL
                )
            """);
            
            // Quest progress table
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS quest_progress (
                    player_uuid TEXT NOT NULL,
                    quest_id TEXT NOT NULL,
                    state TEXT NOT NULL,
                    progress TEXT,
                    started_at INTEGER,
                    completed_at INTEGER,
                    PRIMARY KEY (player_uuid, quest_id)
                )
            """);
            
            // Create indices
            stmt.execute("CREATE INDEX IF NOT EXISTS idx_quest_player ON quest_progress(player_uuid)");
            stmt.execute("CREATE INDEX IF NOT EXISTS idx_quest_id ON quest_progress(quest_id)");
        }
    }
    
    @Override
    public <T> void save(String key, T data) {
        String tableName = getTableName(data.getClass());
        String json = JsonSerializer.serialize(data);
        
        try (PreparedStatement stmt = connection.prepareStatement(
            "INSERT OR REPLACE INTO " + tableName + " (uuid, data, last_updated) VALUES (?, ?, ?)"
        )) {
            stmt.setString(1, key);
            stmt.setString(2, json);
            stmt.setLong(3, System.currentTimeMillis());
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new StorageException("Failed to save data", e);
        }
    }
    
    @Override
    public <T> T load(String key, Class<T> type) {
        String tableName = getTableName(type);
        
        try (PreparedStatement stmt = connection.prepareStatement(
            "SELECT data FROM " + tableName + " WHERE uuid = ?"
        )) {
            stmt.setString(1, key);
            
            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    String json = rs.getString("data");
                    return JsonSerializer.deserialize(json, type);
                }
            }
        } catch (SQLException e) {
            throw new StorageException("Failed to load data", e);
        }
        
        return null;
    }
    
    @Override
    public <T> List<T> query(Query query, Class<T> type) {
        List<T> results = new ArrayList<>();
        String sql = query.toSQL();
        
        try (PreparedStatement stmt = connection.prepareStatement(sql)) {
            query.bindParameters(stmt);
            
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    String json = rs.getString("data");
                    results.add(JsonSerializer.deserialize(json, type));
                }
            }
        } catch (SQLException e) {
            throw new StorageException("Query failed", e);
        }
        
        return results;
    }
    
    private String getTableName(Class<?> type) {
        // Simple implementation - could use annotations
        return type.getSimpleName().toLowerCase() + "_data";
    }
}
```

### 3. MySQL/PostgreSQL Storage

Production-grade relational database storage.

```java
public class MySQLStorage implements StorageProvider {
    private final HikariDataSource dataSource;
    
    public MySQLStorage(DatabaseConfig config) {
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(config.getUrl());
        hikariConfig.setUsername(config.getUsername());
        hikariConfig.setPassword(config.getPassword());
        hikariConfig.setMaximumPoolSize(config.getPoolSize());
        hikariConfig.setConnectionTimeout(config.getConnectionTimeout());
        
        this.dataSource = new HikariDataSource(hikariConfig);
    }
    
    @Override
    public void initialize() {
        try (Connection conn = dataSource.getConnection()) {
            createTables(conn);
        } catch (SQLException e) {
            throw new StorageException("Failed to initialize MySQL", e);
        }
    }
    
    private void createTables(Connection conn) throws SQLException {
        try (Statement stmt = conn.createStatement()) {
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS quest_data (
                    id INT AUTO_INCREMENT PRIMARY KEY,
                    quest_id VARCHAR(255) NOT NULL,
                    name VARCHAR(255) NOT NULL,
                    description TEXT,
                    category VARCHAR(100),
                    data JSON,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                    UNIQUE KEY unique_quest (quest_id)
                )
            """);
            
            stmt.execute("""
                CREATE TABLE IF NOT EXISTS player_quest_progress (
                    id BIGINT AUTO_INCREMENT PRIMARY KEY,
                    player_uuid CHAR(36) NOT NULL,
                    quest_id VARCHAR(255) NOT NULL,
                    state ENUM('UNAVAILABLE', 'AVAILABLE', 'ACTIVE', 'COMPLETED', 'FAILED') NOT NULL,
                    progress JSON,
                    started_at TIMESTAMP NULL,
                    completed_at TIMESTAMP NULL,
                    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                    UNIQUE KEY unique_player_quest (player_uuid, quest_id),
                    INDEX idx_player (player_uuid),
                    INDEX idx_quest (quest_id),
                    INDEX idx_state (state)
                )
            """);
        }
    }
    
    public void saveQuestProgress(UUID playerId, String questId, QuestProgress progress) {
        String sql = """
            INSERT INTO player_quest_progress 
            (player_uuid, quest_id, state, progress, started_at, completed_at)
            VALUES (?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE
            state = VALUES(state),
            progress = VALUES(progress),
            completed_at = VALUES(completed_at)
        """;
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            stmt.setString(1, playerId.toString());
            stmt.setString(2, questId);
            stmt.setString(3, progress.getState().name());
            stmt.setString(4, JsonSerializer.serialize(progress));
            stmt.setTimestamp(5, Timestamp.from(progress.getStartedAt()));
            stmt.setTimestamp(6, progress.getCompletedAt() != null ? 
                Timestamp.from(progress.getCompletedAt()) : null);
            
            stmt.executeUpdate();
        } catch (SQLException e) {
            throw new StorageException("Failed to save quest progress", e);
        }
    }
}
```

### 4. MongoDB Storage

NoSQL document storage for flexible schemas.

```java
public class MongoDBStorage implements StorageProvider {
    private final MongoClient mongoClient;
    private final MongoDatabase database;
    
    public MongoDBStorage(String connectionString, String databaseName) {
        this.mongoClient = MongoClients.create(connectionString);
        this.database = mongoClient.getDatabase(databaseName);
    }
    
    @Override
    public <T> void save(String key, T data) {
        String collectionName = getCollectionName(data.getClass());
        MongoCollection<Document> collection = database.getCollection(collectionName);
        
        Document document = Document.parse(JsonSerializer.serialize(data));
        document.put("_id", key);
        document.put("updated_at", new Date());
        
        collection.replaceOne(
            Filters.eq("_id", key),
            document,
            new ReplaceOptions().upsert(true)
        );
    }
    
    @Override
    public <T> T load(String key, Class<T> type) {
        String collectionName = getCollectionName(type);
        MongoCollection<Document> collection = database.getCollection(collectionName);
        
        Document document = collection.find(Filters.eq("_id", key)).first();
        
        if (document != null) {
            document.remove("_id");
            return JsonSerializer.deserialize(document.toJson(), type);
        }
        
        return null;
    }
    
    @Override
    public <T> List<T> query(Query query, Class<T> type) {
        String collectionName = getCollectionName(type);
        MongoCollection<Document> collection = database.getCollection(collectionName);
        
        List<T> results = new ArrayList<>();
        
        FindIterable<Document> documents = collection.find(query.toBson());
        
        for (Document doc : documents) {
            doc.remove("_id");
            results.add(JsonSerializer.deserialize(doc.toJson(), type));
        }
        
        return results;
    }
    
    private String getCollectionName(Class<?> type) {
        return type.getSimpleName().toLowerCase() + "s";
    }
}
```

---

## Serialization and Deserialization

### JSON Serialization

```java
public class JsonSerializer {
    private static final Gson GSON = new GsonBuilder()
        .setPrettyPrinting()
        .registerTypeAdapter(Location.class, new LocationAdapter())
        .registerTypeAdapter(ItemStack.class, new ItemStackAdapter())
        .create();
    
    public static <T> String serialize(T object) {
        return GSON.toJson(object);
    }
    
    public static <T> T deserialize(String json, Class<T> type) {
        return GSON.fromJson(json, type);
    }
    
    // Custom type adapters
    private static class LocationAdapter implements JsonSerializer<Location>, JsonDeserializer<Location> {
        @Override
        public JsonElement serialize(Location location, Type type, JsonSerializationContext context) {
            JsonObject obj = new JsonObject();
            obj.addProperty("world", location.getWorld().getName());
            obj.addProperty("x", location.getX());
            obj.addProperty("y", location.getY());
            obj.addProperty("z", location.getZ());
            obj.addProperty("yaw", location.getYaw());
            obj.addProperty("pitch", location.getPitch());
            return obj;
        }
        
        @Override
        public Location deserialize(JsonElement json, Type type, JsonDeserializationContext context) {
            JsonObject obj = json.getAsJsonObject();
            World world = getWorld(obj.get("world").getAsString());
            return new Location(
                world,
                obj.get("x").getAsDouble(),
                obj.get("y").getAsDouble(),
                obj.get("z").getAsDouble(),
                obj.get("yaw").getAsFloat(),
                obj.get("pitch").getAsFloat()
            );
        }
    }
}
```

### Binary Serialization

```java
public class BinarySerializer {
    public static <T extends Serializable> byte[] serialize(T object) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            
            oos.writeObject(object);
            return baos.toByteArray();
        } catch (IOException e) {
            throw new SerializationException("Failed to serialize object", e);
        }
    }
    
    public static <T> T deserialize(byte[] bytes, Class<T> type) {
        try (ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
             ObjectInputStream ois = new ObjectInputStream(bais)) {
            
            return type.cast(ois.readObject());
        } catch (IOException | ClassNotFoundException e) {
            throw new SerializationException("Failed to deserialize object", e);
        }
    }
}
```

### NBT Serialization (Minecraft-style)

```java
public class NBTSerializer {
    public static NBTTagCompound serialize(Object object) {
        NBTTagCompound nbt = new NBTTagCompound();
        
        for (Field field : object.getClass().getDeclaredFields()) {
            field.setAccessible(true);
            
            try {
                String name = field.getName();
                Object value = field.get(object);
                
                if (value instanceof String) {
                    nbt.setString(name, (String) value);
                } else if (value instanceof Integer) {
                    nbt.setInt(name, (Integer) value);
                } else if (value instanceof Double) {
                    nbt.setDouble(name, (Double) value);
                } else if (value instanceof Boolean) {
                    nbt.setBoolean(name, (Boolean) value);
                }
                // Add more types as needed
            } catch (IllegalAccessException e) {
                throw new SerializationException("Failed to serialize field: " + field.getName(), e);
            }
        }
        
        return nbt;
    }
    
    public static <T> T deserialize(NBTTagCompound nbt, Class<T> type) {
        try {
            T instance = type.getDeclaredConstructor().newInstance();
            
            for (Field field : type.getDeclaredFields()) {
                field.setAccessible(true);
                String name = field.getName();
                
                if (nbt.hasKey(name)) {
                    Class<?> fieldType = field.getType();
                    
                    if (fieldType == String.class) {
                        field.set(instance, nbt.getString(name));
                    } else if (fieldType == int.class || fieldType == Integer.class) {
                        field.set(instance, nbt.getInt(name));
                    } else if (fieldType == double.class || fieldType == Double.class) {
                        field.set(instance, nbt.getDouble(name));
                    } else if (fieldType == boolean.class || fieldType == Boolean.class) {
                        field.set(instance, nbt.getBoolean(name));
                    }
                }
            }
            
            return instance;
        } catch (Exception e) {
            throw new SerializationException("Failed to deserialize NBT", e);
        }
    }
}
```

---

## Database Integration

### Connection Pool Management

```java
public class DatabaseConnectionPool {
    private final HikariDataSource dataSource;
    
    public DatabaseConnectionPool(DatabaseConfig config) {
        HikariConfig hikariConfig = new HikariConfig();
        
        // Connection settings
        hikariConfig.setJdbcUrl(config.getUrl());
        hikariConfig.setUsername(config.getUsername());
        hikariConfig.setPassword(config.getPassword());
        
        // Pool settings
        hikariConfig.setMaximumPoolSize(config.getMaxPoolSize());
        hikariConfig.setMinimumIdle(config.getMinIdleConnections());
        hikariConfig.setConnectionTimeout(config.getConnectionTimeout());
        hikariConfig.setIdleTimeout(config.getIdleTimeout());
        hikariConfig.setMaxLifetime(config.getMaxLifetime());
        
        // Performance settings
        hikariConfig.setAutoCommit(true);
        hikariConfig.setCachePrepStmts(true);
        hikariConfig.setPrepStmtCacheSize(250);
        hikariConfig.setPrepStmtCacheSqlLimit(2048);
        
        this.dataSource = new HikariDataSource(hikariConfig);
    }
    
    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    public void shutdown() {
        if (dataSource != null && !dataSource.isClosed()) {
            dataSource.close();
        }
    }
}
```

### Repository Pattern

```java
public interface Repository<T, ID> {
    void save(T entity);
    T findById(ID id);
    List<T> findAll();
    void delete(ID id);
    boolean exists(ID id);
}

public class QuestRepository implements Repository<Quest, String> {
    private final StorageProvider storage;
    
    public QuestRepository(StorageProvider storage) {
        this.storage = storage;
    }
    
    @Override
    public void save(Quest quest) {
        storage.save("quest:" + quest.getId(), quest);
    }
    
    @Override
    public Quest findById(String id) {
        return storage.load("quest:" + id, Quest.class);
    }
    
    @Override
    public List<Quest> findAll() {
        Query query = new Query()
            .from("quest_data")
            .orderBy("name", Order.ASC);
        
        return storage.query(query, Quest.class);
    }
    
    @Override
    public void delete(String id) {
        storage.delete("quest:" + id);
    }
    
    @Override
    public boolean exists(String id) {
        return storage.exists("quest:" + id);
    }
    
    // Custom query methods
    public List<Quest> findByCategory(String category) {
        Query query = new Query()
            .from("quest_data")
            .where("category", Operator.EQUALS, category);
        
        return storage.query(query, Quest.class);
    }
    
    public List<Quest> findAvailableQuests(Player player) {
        Query query = new Query()
            .from("quest_data")
            .where("min_level", Operator.LESS_THAN_OR_EQUAL, player.getLevel())
            .where("state", Operator.EQUALS, QuestState.AVAILABLE);
        
        return storage.query(query, Quest.class);
    }
}
```

---

## Player Data Storage

### Player Data Model

```java
public class PlayerData implements Serializable {
    private UUID playerId;
    private String playerName;
    private Set<String> completedQuests;
    private Map<String, QuestProgress> activeQuests;
    private Map<String, Integer> reputation;
    private PlayerStatistics statistics;
    private long lastSeen;
    
    // Getters and setters...
    
    public static class PlayerStatistics {
        private int questsCompleted;
        private int questsFailed;
        private int npcsInteracted;
        private long totalPlaytime;
        
        // Getters and setters...
    }
}
```

### Player Data Manager

```java
public class PlayerDataManager {
    private final StorageProvider storage;
    private final Map<UUID, PlayerData> cache = new ConcurrentHashMap<>();
    
    public PlayerDataManager(StorageProvider storage) {
        this.storage = storage;
    }
    
    public PlayerData loadPlayerData(UUID playerId) {
        // Check cache first
        PlayerData data = cache.get(playerId);
        if (data != null) {
            return data;
        }
        
        // Load from storage
        data = storage.load("player:" + playerId, PlayerData.class);
        
        // Create new if doesn't exist
        if (data == null) {
            data = createNewPlayerData(playerId);
        }
        
        // Cache and return
        cache.put(playerId, data);
        return data;
    }
    
    public void savePlayerData(UUID playerId) {
        PlayerData data = cache.get(playerId);
        if (data != null) {
            data.setLastSeen(System.currentTimeMillis());
            storage.save("player:" + playerId, data);
        }
    }
    
    public void saveAllPlayerData() {
        cache.forEach((playerId, data) -> {
            storage.save("player:" + playerId, data);
        });
    }
    
    public void unloadPlayerData(UUID playerId) {
        savePlayerData(playerId);
        cache.remove(playerId);
    }
    
    private PlayerData createNewPlayerData(UUID playerId) {
        PlayerData data = new PlayerData();
        data.setPlayerId(playerId);
        data.setCompletedQuests(new HashSet<>());
        data.setActiveQuests(new HashMap<>());
        data.setReputation(new HashMap<>());
        data.setStatistics(new PlayerData.PlayerStatistics());
        data.setLastSeen(System.currentTimeMillis());
        return data;
    }
}
```

---

## World Data Storage

### World Data Model

```java
public class WorldData {
    private String worldName;
    private Map<String, NPCData> npcs;
    private Map<String, QuestState> questStates;
    private Set<String> discoveredLocations;
    private long createdAt;
    private long lastModified;
    
    // Getters and setters...
}
```

### World Data Manager

```java
public class WorldDataManager {
    private final StorageProvider storage;
    private final Map<String, WorldData> loadedWorlds = new ConcurrentHashMap<>();
    
    public WorldData loadWorldData(String worldName) {
        return loadedWorlds.computeIfAbsent(worldName, name -> {
            WorldData data = storage.load("world:" + name, WorldData.class);
            
            if (data == null) {
                data = createNewWorldData(name);
            }
            
            return data;
        });
    }
    
    public void saveWorldData(String worldName) {
        WorldData data = loadedWorlds.get(worldName);
        if (data != null) {
            data.setLastModified(System.currentTimeMillis());
            storage.save("world:" + worldName, data);
        }
    }
    
    public void saveAllWorldData() {
        loadedWorlds.forEach((worldName, data) -> {
            storage.save("world:" + worldName, data);
        });
    }
    
    private WorldData createNewWorldData(String worldName) {
        WorldData data = new WorldData();
        data.setWorldName(worldName);
        data.setNpcs(new HashMap<>());
        data.setQuestStates(new HashMap<>());
        data.setDiscoveredLocations(new HashSet<>());
        data.setCreatedAt(System.currentTimeMillis());
        data.setLastModified(System.currentTimeMillis());
        return data;
    }
}
```

---

## Quest Data Persistence

### Quest Progress Tracking

```java
public class QuestProgress implements Serializable {
    private String questId;
    private QuestState state;
    private Map<String, ObjectiveProgress> objectives;
    private Instant startedAt;
    private Instant completedAt;
    private int attempts;
    
    public static class ObjectiveProgress {
        private int current;
        private int target;
        private boolean completed;
        
        public float getProgress() {
            return target > 0 ? (float) current / target : 0;
        }
    }
}
```

### Quest Persistence Manager

```java
public class QuestPersistenceManager {
    private final StorageProvider storage;
    
    public void saveQuestProgress(UUID playerId, String questId, QuestProgress progress) {
        String key = String.format("quest_progress:%s:%s", playerId, questId);
        storage.save(key, progress);
    }
    
    public QuestProgress loadQuestProgress(UUID playerId, String questId) {
        String key = String.format("quest_progress:%s:%s", playerId, questId);
        return storage.load(key, QuestProgress.class);
    }
    
    public Map<String, QuestProgress> loadAllQuestProgress(UUID playerId) {
        Query query = new Query()
            .from("quest_progress")
            .where("player_uuid", Operator.EQUALS, playerId.toString());
        
        List<QuestProgress> progressList = storage.query(query, QuestProgress.class);
        
        return progressList.stream()
            .collect(Collectors.toMap(
                QuestProgress::getQuestId,
                Function.identity()
            ));
    }
    
    public void deleteQuestProgress(UUID playerId, String questId) {
        String key = String.format("quest_progress:%s:%s", playerId, questId);
        storage.delete(key);
    }
}
```

---

## Caching Strategies

### Simple Cache

```java
public class SimpleCache<K, V> {
    private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final long ttl;
    
    public SimpleCache(long ttl) {
        this.ttl = ttl;
    }
    
    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttl));
    }
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        
        if (entry == null) {
            return null;
        }
        
        if (entry.isExpired()) {
            cache.remove(key);
            return null;
        }
        
        return entry.getValue();
    }
    
    public void invalidate(K key) {
        cache.remove(key);
    }
    
    public void clear() {
        cache.clear();
    }
    
    private static class CacheEntry<V> {
        private final V value;
        private final long expiresAt;
        
        public CacheEntry(V value, long expiresAt) {
            this.value = value;
            this.expiresAt = expiresAt;
        }
        
        public V getValue() {
            return value;
        }
        
        public boolean isExpired() {
            return System.currentTimeMillis() > expiresAt;
        }
    }
}
```

### LRU Cache

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public LRUCache(int maxSize) {
        super(16, 0.75f, true);
        this.maxSize = maxSize;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

---

## Migration Strategies

### Schema Migration

```java
public class DatabaseMigration {
    private final Connection connection;
    private final int currentVersion;
    
    public void migrate() throws SQLException {
        int dbVersion = getDatabaseVersion();
        
        while (dbVersion < currentVersion) {
            switch (dbVersion) {
                case 0 -> migrateV0ToV1();
                case 1 -> migrateV1ToV2();
                case 2 -> migrateV2ToV3();
                // Add more migrations...
            }
            dbVersion++;
            setDatabaseVersion(dbVersion);
        }
    }
    
    private void migrateV0ToV1() throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("ALTER TABLE player_data ADD COLUMN reputation JSON");
        }
    }
    
    private void migrateV1ToV2() throws SQLException {
        try (Statement stmt = connection.createStatement()) {
            stmt.execute("""
                CREATE TABLE quest_chains (
                    id INT AUTO_INCREMENT PRIMARY KEY,
                    chain_id VARCHAR(255) NOT NULL UNIQUE,
                    name VARCHAR(255) NOT NULL,
                    quests JSON NOT NULL
                )
            """);
        }
    }
}
```

---

## Performance Optimization

### Batch Operations

```java
public class BatchOperations {
    public void batchSave(List<PlayerData> playerDataList) {
        String sql = "INSERT INTO player_data (uuid, data, last_updated) VALUES (?, ?, ?) " +
                    "ON DUPLICATE KEY UPDATE data = VALUES(data), last_updated = VALUES(last_updated)";
        
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {
            
            for (PlayerData data : playerDataList) {
                stmt.setString(1, data.getPlayerId().toString());
                stmt.setString(2, JsonSerializer.serialize(data));
                stmt.setLong(3, System.currentTimeMillis());
                stmt.addBatch();
            }
            
            stmt.executeBatch();
        } catch (SQLException e) {
            throw new StorageException("Batch save failed", e);
        }
    }
}
```

### Async Operations

```java
public class AsyncStorage {
    private final ExecutorService executor = Executors.newFixedThreadPool(4);
    private final StorageProvider storage;
    
    public CompletableFuture<Void> saveAsync(String key, Object data) {
        return CompletableFuture.runAsync(() -> storage.save(key, data), executor);
    }
    
    public <T> CompletableFuture<T> loadAsync(String key, Class<T> type) {
        return CompletableFuture.supplyAsync(() -> storage.load(key, type), executor);
    }
}
```

---

## Backup and Recovery

### Automatic Backups

```java
public class BackupManager {
    private final Path backupDirectory;
    private final StorageProvider storage;
    
    public void createBackup() {
        String timestamp = new SimpleDateFormat("yyyy-MM-dd_HH-mm-ss").format(new Date());
        Path backupPath = backupDirectory.resolve("backup_" + timestamp);
        
        try {
            Files.createDirectories(backupPath);
            storage.flush();
            
            // Copy data files
            copyDataFiles(storage.getDataDirectory(), backupPath);
            
            // Compress backup
            compressBackup(backupPath);
            
            // Cleanup old backups
            cleanupOldBackups();
        } catch (IOException e) {
            throw new BackupException("Backup failed", e);
        }
    }
    
    private void cleanupOldBackups() {
        // Keep only last 10 backups
        // Implementation...
    }
}
```

---

## Best Practices

### DO:
- ✅ Use connection pooling
- ✅ Implement caching
- ✅ Batch database operations
- ✅ Use async operations where appropriate
- ✅ Validate data before saving
- ✅ Create regular backups
- ✅ Index frequently queried columns

### DON'T:
- ❌ Store sensitive data unencrypted
- ❌ Perform blocking I/O on main thread
- ❌ Forget to close database connections
- ❌ Save data synchronously on every change
- ❌ Ignore migration strategies
- ❌ Skip data validation

---

## Troubleshooting

### Connection Issues

Check connection pool settings, database availability, and credentials.

### Data Corruption

Implement checksums and validation.

### Performance Problems

Profile queries, add indices, use caching.

For more information:
- [Configuration Guide](configuration.md)
- [Quest Development](quest-development.md)
- [API Reference](../api/framework-storage.md)
