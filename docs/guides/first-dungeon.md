# Creating Your First Dungeon

Welcome! This guide will walk you through creating a simple 3-room dungeon using the Argonath Dungeons system.

## Prerequisites

- Argonath Systems installed
- Basic understanding of YAML
- Access to the Prefab Designer (optional, but helpful)

## What You'll Build

A simple crypt dungeon with:
- **Entrance room** - Safe spawn point
- **Combat room** - 3 skeleton enemies
- **Boss room** - Skeleton King boss fight

**Estimated Time:** 30 minutes

---

## Step 1: Create Dungeon Configuration

Create a new file: `plugins/Argonath-Dungeons/config/dungeons/my_first_dungeon.yml`

```yaml
Dungeon:
  # Basic Info
  Id: "my_server:first_crypt"
  Name: "My First Crypt"
  Description: "A small practice dungeon"
  
  # Player Requirements
  Access:
    MinLevel: 1                # No level requirement
    MinPlayers: 1              # Can solo
    MaxPlayers: 3              # Max 3 players
    RequiredKey: null          # No key needed
  
  # Instance Settings
  Instance:
    MaxDuration: 1800          # 30 minutes
    RespawnOnWipe: true        # Players respawn if all die
    RespawnDelay: 10           # 10 second delay
  
  # Difficulty
  Difficulties:
    Normal:
      EnemyHealthMultiplier: 1.0
      EnemyDamageMultiplier: 1.0
      LootTierBonus: 0
  
  # Lockout (none for practice dungeon)
  Lockout:
    Type: NONE
  
  # Generation
  Generation:
    Mode: WFC_GRAPH_GRAMMAR
    WFCRules: "my_server:simple_crypt_rules"
    ProgressionGraph: "my_server:first_crypt_graph"
    PrefabSet: "argonath:crypt_prefabs_v1"
    GridSize: [5, 1, 5]        # Small 5x5 grid
```

---

## Step 2: Define WFC Rules

Create: `config/wfc_rules/simple_crypt_rules.yml`

```yaml
WFCAdjacencyRules:
  RuleSet: "my_server:simple_crypt_rules"
  
  # Simple room types
  RoomTypes:
    WALL:
      Weight: 1.0
    
    CORRIDOR:
      Weight: 2.0
      Description: "Simple corridor"
    
    ROOM:
      Weight: 1.0
      Description: "Generic room"
  
  # Adjacency rules
  Rules:
    # Corridors connect rooms
    CORRIDOR:
      North: [CORRIDOR, ROOM, WALL]
      South: [CORRIDOR, ROOM, WALL]
      East: [WALL]
      West: [WALL]
      Up: [WALL]
      Down: [WALL]
    
    # Rooms can connect to corridors
    ROOM:
      North: [CORRIDOR, WALL]
      South: [CORRIDOR, WALL]
      East: [CORRIDOR, WALL]
      West: [CORRIDOR, WALL]
      Up: [WALL]
      Down: [WALL]
    
    # Walls only connect to walls
    WALL:
      North: [WALL]
      South: [WALL]
      East: [WALL]
      West: [WALL]
      Up: [WALL]
      Down: [WALL]
```

---

## Step 3: Create Progression Graph

Create: `config/dungeon_graphs/first_crypt_graph.yml`

```yaml
ProgressionGraph:
  Id: "my_server:first_crypt_graph"
  Name: "First Crypt Progression"
  
  # Define rooms
  Nodes:
    entrance:
      Type: ENTRANCE
      RoomType: ROOM
      Required: true
      Description: "Safe entrance room"
      PrefabPool: ["argonath:crypt_entrance_01"]
    
    combat:
      Type: MONSTER_ROOM
      RoomType: ROOM
      Description: "Combat encounter"
      Spawners:
        - EntityType: "skeleton"
          Count: 3
          Level: 5
      PrefabPool: ["argonath:crypt_combat_small_01"]
    
    boss:
      Type: BOSS_ARENA
      RoomType: ROOM
      Description: "Boss fight"
      BossId: "my_server:skeleton_king"
      Terminal: true
      PrefabPool: ["argonath:crypt_boss_arena_01"]
  
  # Define progression (entrance → combat → boss)
  Edges:
    - From: entrance
      To: combat
    
    - From: combat
      To: boss
```

---

## Step 4: Create Boss

Create: `config/bosses/skeleton_king.yml`

```yaml
Boss:
  Id: "my_server:skeleton_king"
  Name: "§6Skeleton King"
  Level: 10
  Health: 1000
  Armor: 20
  
  # Visual
  Model: "skeleton"
  Scale: 1.5
  
  # Single phase
  Phases:
    phase1:
      HealthRange: [100, 0]
      
      Abilities:
        # Simple melee attack
        - Id: "slash"
          Type: MELEE
          Cooldown: 3.0
          Damage: 50
          Range: 3
        
        # Occasional area attack
        - Id: "ground_slam"
          Type: CIRCLE_AOE
          Cooldown: 15.0
          Damage: 100
          Radius: 5
          Telegraph:
            Type: "red_circle"
            Duration: 2.0
  
  # Loot
  LootTable: "my_server:skeleton_king_loot"
  GuaranteedDrops:
    - ItemId: "diamond_sword"
      Probability: 0.5
```

---

## Step 5: Configure Loot

Create: `config/loot_tables/skeleton_king_loot.yml`

```yaml
LootTable:
  Id: "my_server:skeleton_king_loot"
  
  Pools:
    - Rolls: 1              # Always drop 1 item
      Entries:
        - Type: "item"
          Item: "iron_sword"
          Weight: 50
        
        - Type: "item"
          Item: "iron_helmet"
          Weight: 30
        
        - Type: "item"
          Item: "diamond_sword"
          Weight: 10        # Rare
    
    - Rolls: 3              # Drop 3 gold coins
      Entries:
        - Type: "currency"
          Currency: "gold"
          Amount: [5, 20]   # 5-20 gold
          Weight: 100
```

---

## Step 6: Create Entry Command

Add command to your plugin or use the built-in command:

```
/dungeon enter my_server:first_crypt
```

**Or via code:**

```java
import com.argonathsystems.mod.dungeons.instance.InstanceManager;
import com.argonathsystems.mod.dungeons.config.InstanceConfig;

public class MyPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        // Register command
        getCommand("enterdungeon").setExecutor((sender, cmd, label, args) -> {
            if (!(sender instanceof Player player)) {
                return false;
            }
            
            // Create solo party
            List<Player> party = List.of(player);
            
            // Load config
            InstanceConfig config = InstanceConfig.load("my_server:first_crypt");
            
            // Create instance
            InstanceManager manager = InstanceManager.getInstance();
            DungeonInstance instance = manager.createInstance(
                config,
                party,
                System.currentTimeMillis()
            );
            
            // Teleport player
            player.teleport(instance.getEntranceLocation());
            player.sendMessage("§aEntering dungeon...");
            
            return true;
        });
    }
}
```

---

## Step 7: Test Your Dungeon

1. **Reload configs:**
   ```
   /argonath reload
   ```

2. **Enter dungeon:**
   ```
   /dungeon enter my_server:first_crypt
   ```

3. **What to test:**
   - ✅ Entrance room spawns correctly
   - ✅ Combat room has 3 skeletons
   - ✅ Boss room has Skeleton King
   - ✅ Boss abilities trigger
   - ✅ Loot drops on kill
   - ✅ Instance cleanup after exit

---

## Common Issues

### "No path found between entrance and boss"

**Cause:** WFC couldn't connect rooms  
**Solution:** Increase grid size in dungeon config:

```yaml
GridSize: [7, 1, 7]  # Was [5, 1, 5]
```

### "WFC Contradiction Exception"

**Cause:** Adjacency rules too restrictive  
**Solution:** Add more allowed neighbors in WFC rules:

```yaml
CORRIDOR:
  North: [CORRIDOR, ROOM, WALL]  # Add ROOM
```

### "Boss doesn't spawn"

**Cause:** Boss config not found  
**Solution:** Check boss ID matches:

```yaml
# In graph:
BossId: "my_server:skeleton_king"

# In boss config:
Id: "my_server:skeleton_king"  # Must match exactly
```

---

## Next Steps

Now that you have a working dungeon, try:

1. **Add a puzzle room**
   - Create locked door
   - Add lever puzzle
   - Reward with key

2. **Increase difficulty**
   - Add Heroic mode
   - More mob spawners
   - Harder boss phase

3. **Custom prefabs**
   - Use Prefab Designer
   - Create unique rooms
   - Add decorations

4. **Quest integration**
   - Create dungeon quest
   - Track boss kills
   - Award achievements

---

## See Also

- [Advanced Dungeon Design](advanced-dungeon-design.md)
- [Boss Fight Mechanics](boss-fight-tutorial.md)
- [Prefab Designer Guide](prefab-designer.md)
- [WFC Algorithm Explained](wfc-explained.md)
