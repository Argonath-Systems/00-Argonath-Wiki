# World Generation System - Developer Guide

**Module:** `02-adapter-hytale` (World Generation Components)  
**Package:** `com.argonathsystems.worldgen`  
**Version:** 0.1.0 (Planned)  
**Dependencies:** `platform-core`, `framework-config`

## Overview

The World Generation System provides a high-fidelity voxel world generation pipeline based on 16-bit Master Mask PNGs, Whittaker biome classification, atmospheric simulation, and procedural city generation using Jigsaw algorithms.

## Table of Contents

- [Architecture](#architecture)
- [Master Mask System](#master-mask-system)
- [Biome Classification](#biome-classification)
- [Road Network Generation](#road-network-generation)
- [City Generation](#city-generation)
- [Wealth-Contextual NPC System](#wealth-contextual-npc-system)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [Integration Points](#integration-points)
- [Performance Considerations](#performance-considerations)

---

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────────┐
│   Hytale Zone/Biome API (V2)           │ ← Adapter Layer
└─────────────────────────────────────────┘
                  ▲
                  │
┌─────────────────────────────────────────┐
│   Master Mask Pipeline                  │
│  ┌─────────────────────────────────┐  │
│  │ 1. MasterMaskReader             │  │ ← 16-bit PNG Parser
│  │ 2. ToroidalMapper               │  │ ← Infinite Wrapping
│  │ 3. MaskTileCache (LRU)          │  │ ← Performance
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
                  ▲
                  │
┌─────────────────────────────────────────┐
│   Biome Simulation                      │
│  ┌─────────────────────────────────┐  │
│  │ 1. WhittakerBiomeSolver         │  │ ← Temp/Humidity → Biome
│  │ 2. LapseRateCalculator          │  │ ← 3D Atmosphere
│  │ 3. BiomeBorderBlender           │  │ ← Edge Dithering
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
                  ▲
                  │
┌─────────────────────────────────────────┐
│   Civil Engineering                     │
│  ┌─────────────────────────────────┐  │
│  │ 1. TerrainAwarePathfinder (A*)  │  │ ← Road Generation
│  │ 2. InfluenceMapRouter           │  │ ← Trade Routes
│  │ 3. BridgeSelector               │  │ ← Water Crossings
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
                  ▲
                  │
┌─────────────────────────────────────────┐
│   Urbanization                          │
│  ┌─────────────────────────────────┐  │
│  │ 1. JigsawCityGenerator          │  │ ← Socket-based Cities
│  │ 2. PrefabSocket                 │  │ ← Connector System
│  │ 3. DistrictGrammar              │  │ ← Wealth Gradients
│  └─────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Dependencies

**Required Frameworks:**
- `03-framework-config` - Configuration loading (YAML)
- `02-framework-core` - Event system, logging

**Not Required But Recommended:**
- `04-framework-condition` - Biome spawn conditions
- `04-framework-npc` - City NPC population
- `05-framework-quest` - City quest givers

---

## Master Mask System

### Concept

A **Master Mask** is a high-resolution 16-bit RGBA PNG that encodes multiple layers of world data:

| Channel | Data Type | Range | Purpose |
|---------|-----------|-------|---------|
| **Red** | Temperature | -50°C to +60°C | Climate simulation |
| **Green** | Humidity | 0% to 100% | Precipitation |
| **Blue** | Elevation | Y=0 to Y=256 | Height map |
| **Alpha** | Bit-Field | 16 bits | Magic zones, civilization density, faction territories |

### Reading Master Masks

```java
import com.argonathsystems.worldgen.mask.MasterMaskReader;

public class WorldGenExample {
    public void initialize() {
        // Load 16-bit PNG
        BufferedImage masterMask = ImageIO.read(
            new File("maps/middle_earth_16k.png")
        );
        
        MasterMaskReader reader = new MasterMaskReader(masterMask);
        
        // Query at world coordinates (thread-safe)
        int worldX = 1000;
        int worldZ = 2000;
        
        float temperature = reader.getTemperature(worldX, worldZ);  // 18.5°C
        float humidity = reader.getHumidity(worldX, worldZ);        // 0.45 (45%)
        int elevation = reader.getElevation(worldX, worldZ);        // Y=85
        
        // Alpha channel bit-field access
        int magicPotential = reader.getMagicPotential(worldX, worldZ);     // 0-15
        int civilizationDensity = reader.getCivilizationDensity(worldX, worldZ); // 0-15
    }
}
```

### Configuration

**File:** `config/worldgen/master_mask.yml`

```yaml
MasterMask:
  # Path to 16-bit RGBA PNG
  InputFile: "maps/orbis_master_mask_16k.png"
  Format: "16bit_RGBA"
  
  # Channel Scaling
  Channels:
    Red:
      Type: "temperature"
      MinValue: -50.0    # °C
      MaxValue: 60.0     # °C
    
    Green:
      Type: "humidity"
      MinValue: 0.0      # 0% precipitation
      MaxValue: 1.0      # 100% precipitation
    
    Blue:
      Type: "elevation"
      MinValue: 0        # Y=0 (bedrock)
      MaxValue: 256      # Y=256 (build limit)
      Scale: 0.00391     # 1/256 for normalization
    
    Alpha:
      Type: "bitfield"
      Bits:
        CivilizationDensity: [0, 3]   # Bits 0-3
        MagicPotential: [4, 7]        # Bits 4-7
        FactionTerritory: [8, 11]     # Bits 8-11
        Reserved: [12, 15]            # Future use
  
  # Toroidal Wrapping (Infinite World)
  Wrapping:
    Enabled: true
    SeamlessEdges: true
    WrapX: true
    WrapZ: true
  
  # Performance Caching
  Cache:
    TileSize: 512               # Pixels per tile
    MaxTilesInMemory: 64        # LRU cache size
    UseOffHeap: true            # DirectByteBuffer
    PreloadRadius: 2            # Tiles around active chunks
```

### Performance Notes

- **Lookup Speed:** < 1 microsecond per coordinate (direct byte access)
- **Memory:** ~8MB per 512x512 tile (16-bit RGBA)
- **Thread Safety:** `MasterMaskReader` is thread-safe via immutable byte buffer
- **Cache Hit Rate:** >95% in normal gameplay (player movement patterns)

---

## Biome Classification

### Whittaker Biome Model

The system uses a **Whittaker Biome Classification** lookup table (256x256 array) for O(1) biome resolution.

```java
import com.argonathsystems.worldgen.biome.WhittakerBiomeSolver;
import com.argonathsystems.worldgen.biome.BiomeType;

public class BiomeExample {
    public BiomeType getBiome(int worldX, int worldZ) {
        WhittakerBiomeSolver solver = WhittakerBiomeSolver.getInstance();
        
        // Get climate from Master Mask
        float temperature = maskReader.getTemperature(worldX, worldZ); // 25°C
        float humidity = maskReader.getHumidity(worldX, worldZ);       // 0.7 (70%)
        
        // Resolve biome (< 100ns)
        BiomeType biome = solver.resolveBiome(temperature, humidity);
        // Result: TROPICAL_RAINFOREST
        
        return biome;
    }
}
```

### 3D Atmospheric Simulation

Implements **lapse rate** (temperature decreases with elevation):

```java
import com.argonathsystems.worldgen.biome.LapseRateCalculator;

public class AtmosphereExample {
    public BiomeType getBiome3D(int worldX, int worldY, int worldZ) {
        // Get base temperature at sea level
        float baseTemp = maskReader.getTemperature(worldX, worldZ); // 25°C
        float humidity = maskReader.getHumidity(worldX, worldZ);
        
        // Apply lapse rate (6.5°C per 1000m elevation)
        float adjustedTemp = LapseRateCalculator.applyLapseRate(
            baseTemp,
            worldY,
            6.5f  // Standard atmospheric lapse rate
        );
        // At Y=200: 25°C - (200/1000 * 6.5) = 23.7°C
        
        return whittakerSolver.resolveBiome(adjustedTemp, humidity);
    }
}
```

**Result:** Tropical jungle at sea level → Temperate forest at Y=100 → Tundra at Y=200

### Biome Configuration

**File:** `config/worldgen/biomes.yml`

```yaml
Biomes:
  TropicalRainforest:
    Id: "argonath:tropical_rainforest"
    WhittakerRange:
      Temperature: [0.8, 1.0]    # 80-100% of max temp
      Humidity: [0.8, 1.0]       # 80-100% humidity
    
    # Block Palette
    Blocks:
      Surface: "hytale:jungle_grass"
      Subsurface: "hytale:dirt"
      Deep: "hytale:stone"
    
    # Vegetation Density
    Vegetation:
      - Type: "jungle_tree_large"
        Density: 0.3             # 30% coverage
        HeightRange: [10, 25]
      
      - Type: "fern"
        Density: 0.5
        HeightRange: [1, 2]
    
    # Micro-Biomes (Noise-based)
    MicroBiomes:
      - Type: "jungle_clearing"
        Probability: 0.05        # 5% of biome area
        Radius: [5, 15]          # Clearing size
  
  SubtropicalDesert:
    Id: "argonath:desert"
    WhittakerRange:
      Temperature: [0.8, 1.0]
      Humidity: [0.0, 0.2]       # Low humidity = desert
    
    Blocks:
      Surface: "hytale:sand"
      Subsurface: "hytale:sandstone"
      Deep: "hytale:stone"
    
    Vegetation:
      - Type: "cactus"
        Density: 0.05
        HeightRange: [2, 5]
    
    MicroBiomes:
      - Type: "oasis"
        Probability: 0.01        # Rare
        Radius: [8, 12]
        RequiresWater: true
```

### Integration with Condition Framework

```java
import com.argonathsystems.framework.condition.Condition;
import com.argonathsystems.framework.condition.ConditionContext;

public class BiomeCondition implements Condition {
    private final BiomeType requiredBiome;
    
    @Override
    public boolean evaluate(ConditionContext context) {
        Location location = context.getPlayer().getLocation();
        BiomeType currentBiome = worldGen.getBiome(
            location.getBlockX(),
            location.getBlockY(),
            location.getBlockZ()
        );
        
        return currentBiome == requiredBiome;
    }
}
```

**Usage in Quests:**
```yaml
Objective:
  Type: "kill_entity"
  Target: "desert_scorpion"
  Count: 10
  Conditions:
    - Type: "biome"
      Biome: "argonath:subtropical_desert"  # Must be in desert
```

---

## Road Network Generation

### Terrain-Aware A* Pathfinding

Roads are generated using **A*** with terrain cost functions:

```java
import com.argonathsystems.worldgen.roads.TerrainAwarePathfinder;
import com.argonathsystems.worldgen.roads.RoadNetwork;

public class RoadExample {
    public void generateRoad(Location cityA, Location cityB) {
        TerrainAwarePathfinder pathfinder = new TerrainAwarePathfinder(
            maskReader,
            biomeResolver
        );
        
        // Configure costs
        pathfinder.setSlopeWeight(2.0f);        // Avoid steep terrain
        pathfinder.setBiomeWeight(1.0f);        // Prefer easy biomes
        pathfinder.setMaxSlope(1);              // 1 block/step max (stairs)
        
        // Generate path
        List<Location> path = pathfinder.findPath(cityA, cityB);
        
        // Build road
        RoadNetwork network = RoadNetwork.getInstance();
        network.buildRoad(path, RoadType.COBBLESTONE);
    }
}
```

### Cost Function

The A* heuristic considers:

| Factor | Weight | Impact |
|--------|--------|--------|
| **Distance** | 1.0 | Base cost |
| **Slope** | 2.0 | Prefer flat terrain |
| **Biome** | 1.0 | Avoid swamps (5x cost), prefer plains (1x) |
| **River Crossing** | 50.0 | High penalty → prefer narrow crossings |
| **Impassable Cliffs** | ∞ | Blocks with slope > 1 are walls |

### Bridge Generation

Automatic bridge placement at river crossings:

```yaml
# config/worldgen/roads.yml
Roads:
  BridgeSelection:
    # River width → Bridge type
    Thresholds:
      - MaxWidth: 5
        BridgeType: "wooden_plank_bridge"
        Prefab: "argonath:bridge_wood_small"
      
      - MaxWidth: 20
        BridgeType: "stone_arch_bridge"
        Prefab: "argonath:bridge_stone_medium"
      
      - MaxWidth: 100
        BridgeType: "suspension_bridge"
        Prefab: "argonath:bridge_suspension_large"
      
      - MaxWidth: 999999
        BridgeType: "ferry_crossing"
        Prefab: "argonath:ferry_station"
  
  # A* Configuration
  Pathfinding:
    MaxIterations: 10000
    HeuristicWeight: 1.2       # Slightly favor heuristic (faster)
    AllowDiagonal: false       # Grid-aligned roads
```

---

## City Generation

### Jigsaw Algorithm

Cities are generated using **socket-based prefab connections**:

```java
import com.argonathsystems.worldgen.city.JigsawCityGenerator;
import com.argonathsystems.worldgen.city.PrefabSocket;

public class CityExample {
    public void generateCity(Location center, int size) {
        JigsawCityGenerator generator = new JigsawCityGenerator();
        
        // Load prefab set
        PrefabSet prefabs = PrefabSet.load("argonath:medieval_city");
        
        // Configure generation
        CityConfig config = new CityConfig.Builder()
            .setCenter(center)
            .setRadius(size)
            .setWealthGradient(true)      // Rich center, poor outskirts
            .setDensityFromMask(true)     // Use Alpha channel
            .setPrefabSet(prefabs)
            .build();
        
        // Generate
        City city = generator.generate(config, seed);
        
        // Place buildings
        city.place(world);
    }
}
```

### Socket Matching

Prefabs connect via **typed sockets**:

```yaml
# prefabs/commercial/shop.yaml
Connectors:
  - Id: "front_door"
    Type: "road_cobble_medium"    # Must match connector type
    Direction: [0, 0, -1]         # North
    Width: 3
    Priority: 10                  # Higher = more likely
    Tags: ["main_street", "commercial"]
```

**Matching Logic:**
1. Find unconnected socket
2. Search nearby prefabs with compatible sockets
3. Filter by:
   - Socket type match (`road_cobble_medium`)
   - Directional alignment (opposite directions)
   - Tag compatibility (both have `commercial`)
4. Select highest priority match
5. Place and connect

### Wealth Gradient

Cities implement **radial wealth distribution**:

```java
public class DistrictGrammar {
    public int getWealthLevel(Location buildingPos, Location cityCenter, int cityRadius) {
        double distance = buildingPos.distance(cityCenter);
        double normalizedDist = distance / cityRadius;  // 0.0 = center, 1.0 = edge
        
        // Inverse wealth (center is richest)
        int wealthLevel = (int) ((1.0 - normalizedDist) * 5);  // 0-5
        
        return Math.clamp(wealthLevel, 0, 5);
    }
}
```

**Prefab Selection:**
```yaml
# City generation filters prefabs by wealth
Metadata:
  WealthLevel: 4    # Rich district building
  
# Only placed in districts with wealth >= 4
```

---

## Wealth-Contextual NPC System

### Overview

NPCs spawned in cities automatically adapt their **behavior, dialogs, inventory, and quests** based on the wealth level (0-5) of their district. This creates authentic economic stratification and contextual storytelling.

### Key Features

| Feature | Implementation | Example |
|---------|----------------|---------|
| **Wealth-Based Behavior** | NPCs have different specializations per wealth tier | Poor smith = repairs; Rich smith = masterwork |
| **Contextual Backstories** | Procedurally generated life stories tied to district | "Trained in royal forges" vs "Learned from father" |
| **Adaptive Inventories** | Trade goods match economic status | Common tools vs enchanted armor |
| **Quest Difficulty Scaling** | Quest complexity increases with wealth | Gather iron vs Deliver to nobles |
| **Pricing Dynamics** | Poor NPCs charge less, nobles charge premium | 0.8x multiplier vs 2.0x multiplier |
| **Reputation Gates** | High-tier NPCs require player prestige | "Nobles only trust reputable couriers" |

### Architecture Flow

```
District Generation (Wealth 0-5)
        ↓
Building Placement (Tagged: smithy, shop, etc.)
        ↓
NPC Template Selection (BuildingTag → NPC Type)
        ↓
Wealth Variant Matching (Wealth Level → Behavior)
        ↓
NPC Instantiation (Name, Dialog, Inventory, Quests)
        ↓
Dynamic Quest Generation (Wealth-appropriate objectives)
```

### Implementation Requirements

**Required Frameworks:**
- `04-framework-npc` - NPC creation and behavior system
- `05-framework-quest` - Quest generation and tracking
- `03-framework-config` - NPC template configurations

**NPC Framework Extensions Needed:**

```java
// NPC.java enhancements
public interface NPC {
    // Existing methods...
    
    // NEW: Wealth-contextual extensions
    void setWealthLevel(int level);
    int getWealthLevel();
    
    void setBackstory(String backstory);
    String getBackstory();
    
    void setSpecialization(String spec);
    String getSpecialization();
    
    void setInventoryQuality(String quality);  // "common", "uncommon", "rare"
    
    void setPriceMultiplier(float multiplier);
    float getPriceMultiplier();
    
    // Quest offering capability
    List<Quest> getAvailableQuests(Player player);
    void registerQuestTemplate(String questType, QuestTemplate template);
}
```

**Quest Framework Extensions Needed:**

```java
// QuestGenerator.java enhancements
public class QuestGenerator {
    // NEW: Context-aware generation
    public Quest generateContextualQuest(NPC giver, Player player) {
        int wealthLevel = giver.getWealthLevel();
        List<String> questTypes = giver.getQuestTypes();
        
        // Select quest type based on wealth and player reputation
        String questType = selectQuestType(questTypes, wealthLevel, player);
        
        // Generate with wealth-appropriate objectives
        return createQuestFromTemplate(questType, giver, wealthLevel);
    }
    
    // Wealth-based objective factories
    public Objective createGatherObjective(int wealthLevel);
    public Objective createDeliveryObjective(int wealthLevel);
    public Objective createCombatObjective(int wealthLevel);
}
```

### Example: Full Wealth-Contextual Smith

**Wealth Level 0-1 (Poor District):**
```
Name: "§7Jonas"
Specialization: "basic_repairs"
Backstory: "A humble blacksmith struggling to make ends meet.
            Learned the trade from their father, who worked
            these same streets. Dreams of one day owning better tools."

Inventory:
  - Iron Ingot (common)
  - Horseshoe (common)
  - Basic Tools (common)
  
Quests:
  - "Iron for the Forge" (Gather 10 iron ingots)
    Reward: 4 horseshoes + 20 copper
    
Dialog: "Welcome, friend. I don't have much, but I do honest work."
```

**Wealth Level 4-5 (Noble District):**
```
Name: "§6Master Ironheart"
Specialization: "masterwork_crafting"
Backstory: "A master smith patronized by nobility. Trained in
            the royal forges of distant kingdoms. Creates bespoke
            weapons and armor for the city's elite."

Inventory:
  - Ornate Longsword (rare, enchanted)
  - Noble Regalia (rare)
  - Enchanted Plate Armor (rare)

Quests:
  - "Deliver Ceremonial Blade to Duke Ravencrest"
    Requirements: Reputation ≥30, Wealth ≥50
    Reward: 100 gold + 10 nobles faction reputation
    
Dialog: "Ah, a visitor. I trust you have the coin to appreciate
         true craftsmanship?"
```

### Configuration

**File:** `config/worldgen/cities.yml`

```yaml
Cities:
  Generation:
    Algorithm: "jigsaw_socket"
    MaxAttempts: 1000           # Per building placement
    
    # Density from Master Mask
    DensityMapping:
      AlphaValue_0_3: 5          # Sparse (hamlet)
      AlphaValue_4_7: 20         # Village
      AlphaValue_8_11: 50        # Town
      AlphaValue_12_15: 100      # City
    
    # District Types
    Districts:
      - Type: "residential"
        WealthRange: [0, 5]      # All wealth levels
        PrefabTags: ["house", "apartment"]
      
      - Type: "commercial"
        WealthRange: [1, 5]      # No poor commercial
        PrefabTags: ["shop", "market", "inn"]
      
      - Type: "industrial"
        WealthRange: [0, 2]      # Poor/working class
        PrefabTags: ["smithy", "tannery", "mill"]
      
      - Type: "noble"
        WealthRange: [4, 5]      # Rich only
        PrefabTags: ["manor", "estate", "palace"]
  
  # NPC Population Configuration
  NPCPopulation:
    Enabled: true
    
    # Wealth-based NPC templates
    NPCTemplates:
      smith:
        BuildingTag: "smithy"
        
        # Behavior varies by wealth level
        WealthVariants:
          - WealthRange: [0, 1]
            Name: "${firstName}"              # Poor: Just first name
            NameColor: "§7"                   # Gray
            DialogTree: "dialog/smith_poor_district"
            Specialization: "basic_repairs"
            InventoryQuality: "common"
            PriceMultiplier: 0.8
            Backstory: "struggling_artisan"
            QuestTypes: ["gather_resources", "repair_tools"]
          
          - WealthRange: [2, 3]
            Name: "${firstName} the Smith"
            NameColor: "§e"                   # Yellow
            DialogTree: "dialog/smith_artisan_district"
            Specialization: "weapons_armor"
            InventoryQuality: "uncommon"
            PriceMultiplier: 1.0
            Backstory: "guild_trained"
            QuestTypes: ["crafting_commission", "gather_rare_materials"]
          
          - WealthRange: [4, 5]
            Name: "Master ${lastName}"
            NameColor: "§6"                   # Gold
            DialogTree: "dialog/smith_noble_district"
            Specialization: "masterwork_crafting"
            InventoryQuality: "rare"
            PriceMultiplier: 2.0
            Backstory: "royal_trained"
            QuestTypes: ["noble_delivery", "prestigious_commission"]
      
      merchant:
        BuildingTag: "shop"
        WealthVariants:
          - WealthRange: [0, 1]
            Name: "${firstName}"
            DialogTree: "dialog/merchant_poor"
            InventorySize: "small"
            GoodQuality: "common"
            PriceMultiplier: 0.9
            QuestTypes: ["debt_collection", "find_customers"]
          
          - WealthRange: [2, 3]
            Name: "${firstName} ${lastName}"
            DialogTree: "dialog/merchant_middle"
            InventorySize: "medium"
            GoodQuality: "uncommon"
            PriceMultiplier: 1.0
            QuestTypes: ["trade_route", "inventory_expansion"]
          
          - WealthRange: [4, 5]
            Name: "Merchant ${lastName}"
            DialogTree: "dialog/merchant_wealthy"
            InventorySize: "large"
            GoodQuality: "rare"
            PriceMultiplier: 1.5
            QuestTypes: ["exotic_goods", "guild_politics"]
    
    # Population density per wealth level
    Density:
      WealthLevel_0: 0.3          # 30% building occupancy (abandoned buildings)
      WealthLevel_1: 0.5
      WealthLevel_2: 0.7
      WealthLevel_3: 0.8
      WealthLevel_4: 0.9
      WealthLevel_5: 1.0          # 100% occupancy in wealthy areas
```

---

## Integration Points

### Framework Config Integration

```java
import com.argonathsystems.framework.config.ConfigFactory;

public class WorldGenPlugin {
    private MasterMaskConfig maskConfig;
    private BiomeConfig biomeConfig;
    
    public void onEnable() {
        ConfigFactory factory = new ConfigFactory(getDataFolder());
        
        // Load configurations
        this.maskConfig = factory.load(
            "worldgen/master_mask.yml",
            MasterMaskConfig.class
        );
        
        this.biomeConfig = factory.load(
            "worldgen/biomes.yml",
            BiomeConfig.class
        );
        
        // Initialize systems
        initializeMasterMask(maskConfig);
        initializeBiomes(biomeConfig);
    }
}
```

### NPC Framework Integration

Spawn NPCs in generated cities with **wealth-contextual behavior**:

```java
import com.argonathsystems.framework.npc.NPCManager;
import com.argonathsystems.framework.npc.NPC;
import com.argonathsystems.framework.npc.NPCBehavior;

public class CityPopulation {
    public void populateCity(City city) {
        NPCManager npcManager = NPCManager.getInstance();
        
        for (Building building : city.getBuildings()) {
            District district = city.getDistrict(building.getLocation());
            int wealthLevel = district.getWealthLevel();  // 0-5
            
            if (building.hasTag("smithy")) {
                // Create wealth-appropriate smith NPC
                NPC smith = createSmithNPC(building, wealthLevel, npcManager);
                smith.spawn();
            }
            else if (building.hasTag("shop")) {
                NPC merchant = createMerchantNPC(building, wealthLevel, npcManager);
                merchant.spawn();
            }
        }
    }
    
    private NPC createSmithNPC(Building building, int wealthLevel, NPCManager npcManager) {
        NPCBehavior behavior = buildSmithBehavior(wealthLevel);
        String dialogTree = selectSmithDialogTree(wealthLevel);
        String backstory = generateSmithBackstory(wealthLevel);
        
        return npcManager.createNPC("smith")
            .setLocation(building.getEntrance())
            .setName(generateSmithName(wealthLevel))
            .setDialogTree(dialogTree)
            .setBehavior(behavior)
            .setBackstory(backstory)
            .setWealthLevel(wealthLevel)
            .build();
    }
    
    private NPCBehavior buildSmithBehavior(int wealthLevel) {
        NPCBehavior.Builder builder = new NPCBehavior.Builder();
        
        // Wealth-based inventory and specialization
        switch (wealthLevel) {
            case 0, 1: // Poor district smith
                builder.setSpecialization("basic_repairs")
                       .setInventoryQuality("common")
                       .addTradeItem("iron_ingot", 1.0f)
                       .addTradeItem("horseshoe", 0.8f)
                       .addTradeItem("basic_tools", 0.6f)
                       .setPriceMultiplier(0.8f);  // Cheaper prices
                break;
                
            case 2, 3: // Middle-class smith
                builder.setSpecialization("weapons_armor")
                       .setInventoryQuality("uncommon")
                       .addTradeItem("steel_sword", 0.7f)
                       .addTradeItem("chainmail", 0.5f)
                       .addTradeItem("iron_ingot", 1.0f)
                       .setPriceMultiplier(1.0f);  // Standard prices
                break;
                
            case 4, 5: // Wealthy district smith
                builder.setSpecialization("masterwork_crafting")
                       .setInventoryQuality("rare")
                       .addTradeItem("ornate_longsword", 0.4f)
                       .addTradeItem("enchanted_armor", 0.3f)
                       .addTradeItem("noble_regalia", 0.2f)
                       .setPriceMultiplier(2.0f);  // Premium prices
                break;
        }
        
        return builder.build();
    }
    
    private String selectSmithDialogTree(int wealthLevel) {
        // Different dialog trees based on wealth/status
        return wealthLevel >= 4 
            ? "dialog/smith_noble_district" 
            : wealthLevel >= 2
                ? "dialog/smith_artisan_district"
                : "dialog/smith_poor_district";
    }
    
    private String generateSmithBackstory(int wealthLevel) {
        // Contextual backstory generation
        if (wealthLevel <= 1) {
            return "A humble blacksmith struggling to make ends meet. " +
                   "Learned the trade from their father, who worked these same streets. " +
                   "Dreams of one day owning better tools.";
        } else if (wealthLevel <= 3) {
            return "A skilled artisan with a growing reputation. " +
                   "Served an apprenticeship with the city's armorers guild. " +
                   "Known for reliable craftsmanship at fair prices.";
        } else {
            return "A master smith patronized by nobility. " +
                   "Trained in the royal forges of distant kingdoms. " +
                   "Creates bespoke weapons and armor for the city's elite.";
        }
    }
    
    private String generateSmithName(int wealthLevel) {
        // Wealth-appropriate naming (poor = common names, rich = titles)
        if (wealthLevel >= 4) {
            return "§6Master " + NameGenerator.generateNobleSmithName();
        } else if (wealthLevel >= 2) {
            return "§e" + NameGenerator.generateArtisanName() + " the Smith";
        } else {
            return "§7" + NameGenerator.generateCommonName();
        }
    }
}
```

#### Wealth-Based Dialog Examples

**Poor District Smith (Wealth 0-1):**
```yaml
# dialog/smith_poor_district.yml
Greeting:
  Text: "Welcome, friend. I don't have much, but I do honest work."
  
Trades:
  - Text: "I need iron to keep working. Can you spare some?"
    Type: "buy"
    Item: "iron_ingot"
    Price: 5
    
  - Text: "I can repair your tools... for a small fee."
    Type: "service"
    Service: "repair"
    Quality: "basic"
    
QuestOffers:
  - Id: "poor_smith_iron_quest"
    Condition: "reputation >= 0"
    Dialog: "The merchant won't extend my credit anymore. Could you bring me 10 iron ingots? I'll make you horseshoes in return."
```

**Wealthy District Smith (Wealth 4-5):**
```yaml
# dialog/smith_noble_district.yml
Greeting:
  Text: "Ah, a visitor. I trust you have the coin to appreciate true craftsmanship?"
  
Trades:
  - Text: "I have an ornate longsword commissioned for Lord Blackwood. Perhaps you're interested?"
    Type: "sell"
    Item: "ornate_longsword"
    Price: 500
    Requirements:
      - "reputation >= 50"
      - "wealth >= 100"
      
QuestOffers:
  - Id: "noble_smith_delivery_quest"
    Condition: "reputation >= 30 && wealth >= 50"
    Dialog: "I've finished Duke Ravencrest's ceremonial blade. Would you deliver it to his estate? I'll pay handsomely - 100 gold - and you'll earn the Duke's favor."
```

### Quest Framework Integration

Generate **wealth-contextual quests** from NPCs using the generic `QuestGenerator`:

```java
import com.lordofthetales.framework.quest.model.QuestDefinition;
import com.lordofthetales.framework.quest.model.QuestReward;
import com.lordofthetales.framework.quest.model.QuestObjectiveReference;
import com.lordofthetales.framework.quest.service.QuestGenerator;

/**
 * Example: Mod-level wealth-based quest generation.
 * 
 * The QuestGenerator framework provides generic building blocks.
 * Domain-specific logic (wealth tiers, NPC types) lives in your mod.
 */
public class WealthBasedQuestGenerator {
    
    private final QuestGenerator generator;
    
    public WealthBasedQuestGenerator() {
        this.generator = new QuestGenerator();
        registerWealthStrategies();
        registerQuestTemplates();
    }
    
    /**
     * Register generation strategies for different wealth tiers.
     * Strategies encapsulate the domain logic for quest configuration.
     */
    private void registerWealthStrategies() {
        // Poor tier (0-1): Resource gathering, basic rewards
        generator.registerStrategy("poor_smith", (ctx, builder) -> {
            String npcId = ctx.getString("npc_id");
            String npcName = ctx.getString("npc_name", "the smith");
            
            builder.id("poor_smith_iron_" + npcId)
                   .title("Iron for the Forge")
                   .description(npcName + " needs raw materials to continue working.")
                   .type(QuestDefinition.QuestType.SIDE)
                   .addStage(stage -> stage
                       .description("Gather iron ingots")
                       .addObjective(new QuestObjectiveReference("gather_iron", 10)))
                   .addReward(createCurrencyReward("copper", 20));
        });
        
        // Middle tier (2-3): Crafting commissions
        generator.registerStrategy("artisan_smith", (ctx, builder) -> {
            String npcId = ctx.getString("npc_id");
            String npcName = ctx.getString("npc_name", "the artisan");
            
            builder.id("artisan_commission_" + npcId)
                   .title("Commission: Fine Weapon")
                   .description(npcName + " will craft a weapon if you provide materials.")
                   .type(QuestDefinition.QuestType.SIDE)
                   .addStage(stage -> stage
                       .description("Provide crafting materials")
                       .addObjective(new QuestObjectiveReference("gather_steel", 5))
                       .addObjective(new QuestObjectiveReference("gather_leather", 2)))
                   .addReward(createCurrencyReward("silver", 50));
        });
        
        // Wealthy tier (4-5): Delivery/prestige quests
        generator.registerStrategy("noble_smith", (ctx, builder) -> {
            String npcId = ctx.getString("npc_id");
            String npcName = ctx.getString("npc_name", "Master Smith");
            
            builder.id("noble_delivery_" + npcId)
                   .title("Deliver Ceremonial Blade")
                   .description("Master " + npcName + " has crafted an ornate weapon for nobility.")
                   .type(QuestDefinition.QuestType.SIDE)
                   .addStage(stage -> stage
                       .description("Deliver the blade to the noble estate")
                       .addObjective(new QuestObjectiveReference("deliver_blade", 1)))
                   .addReward(createCurrencyReward("gold", 100))
                   .addReward(createReputationReward("nobles_faction", 10));
        });
    }
    
    /**
     * Register reusable quest templates with parameter placeholders.
     */
    private void registerQuestTemplates() {
        generator.registerTemplate("gather_for_npc", template -> template
            .title("Gather {item_name} for {npc_name}")
            .description("{npc_name} needs your help collecting {item_name}.")
            .type(QuestDefinition.QuestType.SIDE)
            .addStage(stage -> stage
                .description("Collect the requested items")
                .addObjective(new QuestObjectiveReference("gather", 1)))
        );
        
        generator.registerTemplate("delivery", template -> template
            .title("Deliver {item_name} to {recipient}")
            .description("Transport {item_name} safely to {recipient}.")
            .type(QuestDefinition.QuestType.SIDE)
            .addStage(stage -> stage
                .addObjective(new QuestObjectiveReference("deliver", 1)))
        );
    }
    
    /**
     * Generate a quest appropriate for the smith's wealth level.
     */
    public QuestDefinition generateSmithQuest(NPC smith, int wealthLevel) {
        String strategyId = switch (wealthLevel) {
            case 0, 1 -> "poor_smith";
            case 2, 3 -> "artisan_smith";
            default -> "noble_smith";
        };
        
        return generator.generate(strategyId, Map.of(
            "npc_id", smith.getId(),
            "npc_name", smith.getName()
        ));
    }
    
    /**
     * Generate from template with custom parameters.
     */
    public QuestDefinition generateGatherQuest(NPC npc, String itemName, int count) {
        return generator.fromTemplate("gather_for_npc")
            .param("npc_name", npc.getName())
            .param("item_name", itemName)
            .build();
    }
    
    // Helper methods for reward creation
    private QuestReward createCurrencyReward(String currency, int amount) {
        QuestReward reward = new QuestReward();
        reward.setType(QuestReward.RewardType.CURRENCY);
        reward.setId(currency);
        reward.setAmount(amount);
        return reward;
    }
    
    private QuestReward createReputationReward(String faction, int amount) {
        QuestReward reward = new QuestReward();
        reward.setType(QuestReward.RewardType.REPUTATION);
        reward.setId(faction);
        reward.setAmount(amount);
        return reward;
    }
}
```

#### Quest Template Examples

**Poor District Quest:**
```yaml
# quests/generated/poor_smith_iron.yml
Quest:
  Id: "poor_smith_iron_${npcId}"
  Name: "Iron for the Forge"
  Giver: "${npcId}"
  
  Backstory: |
    ${npcName} is a struggling blacksmith in the poor district.
    The local mine won't give credit anymore, and without iron,
    the forge goes cold. Help keep an honest worker employed.
  
  Objectives:
    - Type: "gather"
      Item: "iron_ingot"
      Count: 10
      Description: "Mine iron ore or purchase from merchants"
  
  Rewards:
    Items:
      - Type: "horseshoe"
        Count: 4
    Currency:
      - Type: "copper"
        Amount: 20
    Reputation:
      - Faction: "poor_district"
        Amount: 5
  
  Difficulty: "easy"
  EstimatedTime: "15 minutes"
```

**Wealthy District Quest:**
```yaml
# quests/generated/noble_smith_delivery.yml
Quest:
  Id: "noble_smith_delivery_${npcId}"
  Name: "Deliver Ceremonial Blade to ${nobleName}"
  Giver: "${npcId}"
  
  Backstory: |
    Master ${npcName} has completed a masterwork commission
    for ${nobleName}, one of the city's most influential nobles.
    The blade must be delivered with utmost care - this is a 
    matter of prestige and honor.
  
  Objectives:
    - Type: "delivery"
      Item: "ornate_ceremonial_blade"
      Destination: "${nobleManorLocation}"
      Recipient: "${nobleNpcId}"
      FailConditions:
        - "player_death"           # Lose item = fail quest
        - "item_damaged"           # Don't fight with it!
        - "time_limit_exceeded"    # 2 hour deadline
  
  Requirements:
    Conditions:
      - Type: "reputation"
        Faction: "nobles_faction"
        Minimum: 30
        Reason: "Nobles only trust reputable couriers"
      
      - Type: "wealth"
        Minimum: 50
        Reason: "Must appear presentable to enter noble estates"
  
  Rewards:
    Currency:
      - Type: "gold"
        Amount: 100
    Reputation:
      - Faction: "nobles_faction"
        Amount: 10
      - Faction: "${smithGuild}"
        Amount: 5
    
    # Unlock future high-tier quests
    Unlocks:
      - "noble_faction_quest_chain"
  
  Difficulty: "hard"
  EstimatedTime: "30 minutes"
  TimeLimit: "2 hours"
```

---

## Performance Considerations

### Memory Usage

| Component | Memory | Notes |
|-----------|--------|-------|
| Master Mask (16k x 16k) | ~1GB | Full resolution in RAM |
| Tile Cache (64 tiles) | ~512MB | LRU cache of 512x512 tiles |
| Biome LUT | ~256KB | 256x256 array |
| Road Network | ~10MB/1000 roads | Path coordinates |
| City (50 buildings) | ~5MB | Prefab instances |

**Total:** ~1.5GB for large world

### Optimization Strategies

1. **Lazy Loading:** Load Master Mask tiles on-demand
2. **Off-Heap Storage:** Use `DirectByteBuffer` for tile cache
3. **Chunk-Aligned Generation:** Generate roads/cities aligned to Minecraft chunks
4. **Async Generation:** Generate cities on async threads, place on main thread
5. **Prefab Pooling:** Reuse prefab instances with transforms

### Threading Model

```java
// Safe: Master Mask reads (immutable)
CompletableFuture.supplyAsync(() -> {
    return maskReader.getTemperature(x, z);
}, asyncExecutor);

// Unsafe: Biome writes (must be main thread)
Bukkit.getScheduler().runTask(plugin, () -> {
    world.setBiome(x, z, biome);
});
```

---

## API Reference

### MasterMaskReader

```java
package com.argonathsystems.worldgen.mask;

public class MasterMaskReader {
    /**
     * Constructs reader from 16-bit RGBA PNG.
     * 
     * @param image BufferedImage with TYPE_USHORT_GRAY or compatible
     * @throws IllegalArgumentException if not 16-bit
     */
    public MasterMaskReader(BufferedImage image);
    
    /**
     * Gets temperature at world coordinates.
     * 
     * @param x World X coordinate
     * @param z World Z coordinate
     * @return Temperature in Celsius (-50.0 to +60.0)
     * @thread-safe Yes
     */
    public float getTemperature(int x, int z);
    
    /**
     * Gets humidity at world coordinates.
     * 
     * @param x World X coordinate
     * @param z World Z coordinate
     * @return Humidity as fraction (0.0 to 1.0)
     * @thread-safe Yes
     */
    public float getHumidity(int x, int z);
    
    /**
     * Gets elevation at world coordinates.
     * 
     * @param x World X coordinate
     * @param z World Z coordinate
     * @return Y-level (0 to 256)
     * @thread-safe Yes
     */
    public int getElevation(int x, int z);
    
    /**
     * Gets magic potential from Alpha channel bits 4-7.
     * 
     * @param x World X coordinate
     * @param z World Z coordinate
     * @return Magic level (0 to 15)
     * @thread-safe Yes
     */
    public int getMagicPotential(int x, int z);
    
    /**
     * Gets civilization density from Alpha channel bits 0-3.
     * 
     * @param x World X coordinate
     * @param z World Z coordinate
     * @return Density level (0 to 15)
     * @thread-safe Yes
     */
    public int getCivilizationDensity(int x, int z);
}
```

### WhittakerBiomeSolver

```java
package com.argonathsystems.worldgen.biome;

public class WhittakerBiomeSolver {
    /**
     * Gets singleton instance.
     * 
     * @return Global solver instance
     */
    public static WhittakerBiomeSolver getInstance();
    
    /**
     * Resolves biome from climate parameters.
     * 
     * @param temperature Temperature in Celsius
     * @param humidity Humidity as fraction (0.0-1.0)
     * @return Biome type
     * @complexity O(1) - lookup table
     * @thread-safe Yes
     */
    public BiomeType resolveBiome(float temperature, float humidity);
    
    /**
     * Initializes lookup table from configuration.
     * 
     * @param config Biome configuration
     * @throws IllegalStateException if already initialized
     */
    public void initialize(BiomeConfig config);
}
```

### TerrainAwarePathfinder

```java
package com.argonathsystems.worldgen.roads;

public class TerrainAwarePathfinder {
    /**
     * Finds optimal path between two points.
     * 
     * @param start Start location
     * @param end End location
     * @return List of waypoints, or empty if no path
     * @complexity O(n log n) where n = search space
     * @thread-safe No - create new instance per thread
     */
    public List<Location> findPath(Location start, Location end);
    
    /**
     * Sets slope cost multiplier.
     * 
     * @param weight Cost multiplier (default: 2.0)
     */
    public void setSlopeWeight(float weight);
    
    /**
     * Sets maximum traversable slope.
     * 
     * @param maxSlope Max Y-difference per step (default: 1)
     */
    public void setMaxSlope(int maxSlope);
}
```

---

## See Also

- [Dungeon Generation System](dungeon-generation.md)
- [Prefab Designer Integration](prefab-integration.md)
- [Configuration Guide](../guides/configuration.md)
- [Performance Tuning](../advanced/performance-tuning.md)
