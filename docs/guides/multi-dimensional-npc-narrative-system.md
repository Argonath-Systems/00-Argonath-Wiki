# Multi-Dimensional NPC Narrative System

**Version:** 1.0.0  
**Last Updated:** 2026-01-27  
**Status:** ✅ Implemented & Tested

---

## Overview

The **Multi-Dimensional NPC Narrative System** generates rich, contextual NPC backstories and names based on five narrative axes: **Wealth**, **Education**, **Origin**, **Experience**, and **Personality Traits**. This system enables the creation of diverse, believable NPCs that reflect their economic status, educational background, cultural origin, professional experience, and unique character traits.

---

## System Architecture

### Core Components

1. **NPCNarrativeDimensions** ([NPCNarrativeDimensions.java](../../../04-framework-npc/src/main/java/com/argonathsystems/framework/npc/narrative/NPCNarrativeDimensions.java))
   - Multi-dimensional character model with 5 narrative axes
   - Fluent API for setting dimensions
   - Trait management system (add/remove/has)

2. **NPCNarrativeGenerator** ([NPCNarrativeGenerator.java](../../../04-framework-npc/src/main/java/com/argonathsystems/framework/npc/narrative/NPCNarrativeGenerator.java))
   - Template-based backstory generation
   - Contextual narrative variation across all dimensions
   - Deterministic generation with seed-based randomization

3. **NPCNameGenerator** ([NPCNameGenerator.java](../../../04-framework-npc/src/main/java/com/argonathsystems/framework/npc/narrative/NPCNameGenerator.java))
   - Wealth-based name generation with Minecraft color codes
   - Name assembly logic combining first names, surnames, and titles
   - Deterministic generation for consistency

---

## Narrative Dimensions

### 1. Wealth (0-5)

- **0 (Destitute)**: §7 Gray - Struggling to survive
- **1 (Poor)**: §7 Gray - Working class, hard times
- **2 (Middle)**: §f White - Modest living, steady work
- **3 (Comfortable)**: §e Yellow - Prosperous, growing reputation
- **4 (Wealthy)**: §6 Gold - Affluent, serves nobility
- **5 (Noble)**: §b Aqua - Elite, legendary craftsmanship

**Impact:**
- Name color formatting (§7 → §b)
- Price multipliers (0.6x → 2.5x)
- Inventory quality (COMMON → LEGENDARY)
- Backstory circumstances (struggling → prestigious)

### 2. Education

```java
public enum EducationLevel {
    NONE,        // No formal training
    BASIC,       // Learned from observation
    APPRENTICE,  // Formal apprenticeship
    SCHOLAR,     // Guildhall training
    MASTER       // Legendary craftsmen mentorship
}
```

**Impact:**
- Narrative phrases: "learned through necessity" → "mastered under legendary craftsmen"
- Quest complexity and types
- Dialog tree sophistication
- Knowledge-based quest requirements

### 3. Origin

```java
public enum OriginType {
    CITY,     // Urban upbringing
    VILLAGE,  // Small hamlet background
    RURAL,    // Farming family
    NOMADIC,  // Traveling traders
    FOREIGN   // Foreign territories
}
```

**Impact:**
- Backstory opening phrases: "City-born" → "Journeyed here from foreign lands"
- Name surname patterns (city surnames vs. location-based)
- Cultural references in dialog
- Trade item specializations

### 4. Experience

```java
public enum ExperienceLevel {
    NOVICE,     // Just starting out
    JOURNEYMAN, // Competent practitioner
    VETERAN,    // Seasoned professional
    LEGENDARY   // Renowned master
}
```

**Impact:**
- Backstory credibility phrases
- Quest difficulty scaling
- Price multiplier modifiers
- Reputation in community

### 5. Personality Traits (24 Variants)

```java
// Work Ethic
HARDWORKING, LAZY

// Ambition
AMBITIOUS, CONTENT

// Demeanor
FRIENDLY, RECLUSIVE, GRUFF, CAUTIOUS

// Character
TRUSTWORTHY, GREEDY, HONEST, DECEPTIVE, GENEROUS, STINGY

// Specialty
PERFECTIONIST, SLOPPY, INNOVATIVE, TRADITIONAL

// Social
STORYTELLER, QUIET, GOSSIP

// Status
STRUGGLING, PRESTIGIOUS, AMBITIOUS_CLIMBER

// Lifestyle
WANDERER, HOMEBODY
```

**Impact:**
- Backstory flavor phrases
- Dialog tone and style
- Trade negotiation behavior
- Quest objective preferences

---

## Usage Examples

### Example 1: Poor City Smith

```java
NPCNarrativeDimensions dims = new NPCNarrativeDimensions()
    .setWealthLevel(1)
    .setEducation(EducationLevel.APPRENTICE)
    .setOrigin(OriginType.CITY)
    .setExperience(ExperienceLevel.JOURNEYMAN)
    .addTrait(PersonalityTrait.HARDWORKING)
    .addTrait(PersonalityTrait.STRUGGLING);

NPCNarrativeGenerator generator = new NPCNarrativeGenerator(12345L);
String backstory = generator.generateBackstory(dims, "smith", 123L);

NPCNameGenerator nameGen = new NPCNameGenerator(67890L);
String name = nameGen.generateName(dims.getWealthLevel(), dims.getOrigin(), "smith", 456L);
```

**Output:**
- **Name:** `§7Eamon the Smith` (gray color, trade surname)
- **Backstory:** *"City-born and bred in a working-class neighborhood learned smith from their father, serving a proper apprenticeship. Times are hard – creditors circle and materials run scarce. Days start early and end late at the forge."*

### Example 2: Wealthy Noble Smith

```java
NPCNarrativeDimensions dims = new NPCNarrativeDimensions()
    .setWealthLevel(5)
    .setEducation(EducationLevel.MASTER)
    .setOrigin(OriginType.CITY)
    .setExperience(ExperienceLevel.LEGENDARY)
    .addTrait(PersonalityTrait.PERFECTIONIST)
    .addTrait(PersonalityTrait.PRESTIGIOUS);

String backstory = generator.generateBackstory(dims, "smith", 789L);
String name = nameGen.generateName(5, OriginType.CITY, "smith", 999L);
```

**Output:**
- **Name:** `§bGrandmaster Goldtouch` (aqua color, prestigious title, descriptive surname)
- **Backstory:** *"City-born and bred in a noble neighborhood mastered smith under legendary craftsmen at the royal forges. Only the elite can afford such legendary craftsmanship. Reputation is guarded as fiercely as any treasure."*

### Example 3: Nomadic Merchant

```java
NPCNarrativeDimensions dims = new NPCNarrativeDimensions()
    .setWealthLevel(2)
    .setEducation(EducationLevel.BASIC)
    .setOrigin(OriginType.NOMADIC)
    .setExperience(ExperienceLevel.VETERAN)
    .addTrait(PersonalityTrait.WANDERER)
    .addTrait(PersonalityTrait.STORYTELLER);

String backstory = generator.generateBackstory(dims, "merchant", 555L);
String name = nameGen.generateName(2, OriginType.NOMADIC, "merchant", 666L);
```

**Output:**
- **Name:** `§fJonas Shadowvale` (white color, location-based surname)
- **Backstory:** *"Grew up on the road with modest traveling folk picked up merchant from local craftsmen, learning the basics through hard work. Known for honest work and reasonable rates in the community. Tales of distant lands enrich conversations with customers."*

---

## Name Generation

### Name Pools

#### First Names (Common)
Aric, Cael, Darian, Eamon, Finn, Gareth, Hale, Ian, Jorin, Kael, Liam, Maren, Nolan, Oren, Quinn, Rowan, Sean, Torin, Uric, Valen, Wren, Xander, Yorick, Zane, Bran, Drake

#### First Names (Noble - Wealth 4+)
Aldric, Branwell, Cedric, Dorian, Edmund, Finnegan, Gideon, Harlan, Ignatius, Jareth, Kendrick, Leopold, Mortimer, Nathaniel, Oswald, Percival, Quinton, Reginald, Sebastian, Thaddeus, Ulric, Vincent

#### Surnames (Trade-based - Wealth 0-2)
- **Profession Surnames:** Smith, Wright, Cooper, Fletcher, Weaver, Mason, Brewer, Fisher, Tanner, Carter, Baker, Miller

#### Surnames (Descriptive - Wealth 3-5)
- **Descriptive:** Stonebreaker, Ironheart, Swifthand, Goldtouch, Truehammer, Brightspark, Steelheart, Stormforge, Shadowmend, Firehand

#### Surnames (Location-based - All Wealth Levels)
- **Locations:** Ashford, Blackwood, Clearwater, Darkholme, Elmwood, Fairhaven, Greenvale, Highcrest, Irondale, Shadowvale

### Title System (Wealth 4-5)

- **Wealth 4:** "Lord", "Lady"
- **Wealth 5:** "Grandmaster", "Master", "Lord", "Lady"

### Color Code Mapping

| Wealth | Color Code | Color Name | Usage |
|--------|-----------|------------|-------|
| 0-1    | §7        | Gray       | Poor/struggling NPCs |
| 2      | §f        | White      | Middle-class NPCs |
| 3      | §e        | Yellow     | Comfortable NPCs |
| 4      | §6        | Gold       | Wealthy NPCs |
| 5      | §b        | Aqua       | Noble NPCs |

---

## Backstory Template Structure

### Origin Narrative
```
{ORIGIN_PHRASE} + {WEALTH_DESCRIPTOR} + {LOCATION_TYPE}
```

**Examples:**
- *"City-born and bred in a working-class neighborhood"*
- *"Born in the city's noble district"*
- *"Grew up in a small modest hamlet"*
- *"Traveled with wealthy nomadic traders"*

### Education Narrative
```
{EDUCATION_VERB} + {PROFESSION} + {EDUCATION_PHRASE}
```

**Examples:**
- *"learned smith through necessity and observation, without formal training"*
- *"learned smith from their father, serving a proper apprenticeship"*
- *"studied smith at the guildhall, combining practical work with theoretical knowledge"*
- *"mastered smith under legendary craftsmen at prestigious workshops"*

### Circumstances Narrative (Wealth-based)
```
{WEALTH_SPECIFIC_CIRCUMSTANCES}
```

**Poor (0-1):**
- *"Bills pile up, but they refuse to compromise on quality."*
- *"Survival is a daily struggle, yet pride in craft remains."*
- *"Times are hard – creditors circle and materials run scarce."*

**Middle (2-3):**
- *"A small but loyal clientele keeps the forge busy year-round."*
- *"Known for honest work and reasonable rates in the community."*
- *"Business is steady, with a growing reputation among locals."*

**Wealthy (4-5):**
- *"Commissioned exclusively by nobility, creating works of art."*
- *"Only the elite can afford such legendary craftsmanship."*
- *"Patrons wait months for a single custom commission."*

### Personality Narrative
```
{TRAIT_SPECIFIC_FLAVOR_TEXT}
```

**Examples:**
- *"Each day brings new challenges to overcome."* (STRUGGLING)
- *"Tales of distant lands enrich conversations with customers."* (STORYTELLER, WANDERER)
- *"Reputation is guarded as fiercely as any treasure."* (PRESTIGIOUS)
- *"Focus remains on function over ornament."* (PRACTICAL)

---

## Integration with NPC Framework

### Complete NPC Creation Flow

```java
// Step 1: Define dimensions
NPCNarrativeDimensions dims = new NPCNarrativeDimensions()
    .setWealthLevel(3)
    .setEducation(EducationLevel.APPRENTICE)
    .setOrigin(OriginType.CITY)
    .setExperience(ExperienceLevel.VETERAN)
    .addTrait(PersonalityTrait.TRUSTWORTHY)
    .addTrait(PersonalityTrait.PRACTICAL);

// Step 2: Generate backstory
NPCNarrativeGenerator generator = new NPCNarrativeGenerator(seed);
String backstory = generator.generateBackstory(dims, "blacksmith", npcSeed);

// Step 3: Generate name
NPCNameGenerator nameGen = new NPCNameGenerator(seed);
String coloredName = nameGen.generateName(dims.getWealthLevel(), dims.getOrigin(), "blacksmith", npcSeed);
String plainName = nameGen.generatePlainName(dims.getWealthLevel(), dims.getOrigin(), "blacksmith", npcSeed);

// Step 4: Create behavior
NPCBehavior behavior = NPCBehavior.builder()
    .setInventoryQuality(NPCBehavior.InventoryQuality.RARE)
    .setPriceMultiplier(1.2f)
    .addTradeItem("iron_ingot", 0.8f)
    .addTradeItem("steel_ingot", 0.5f)
    .build();

// Step 5: Build NPC
NPC npc = NPCManager.getInstance()
    .createNPC(plainName)
    .setWealthLevel(dims.getWealthLevel())
    .setBackstory(backstory)
    .setSpecialization("blacksmith")
    .setBehavior(behavior)
    .build();
```

---

## Testing

### Test Coverage

**Unit Tests (17 tests):**
- ✅ NPCNarrativeGeneratorTest: Backstory generation for all dimensions
- ✅ NPCNameGeneratorTest: Name generation with color codes

**Integration Tests (4 tests):**
- ✅ NPCCreationIntegrationTest: Full NPC creation flow
- ✅ Poor city smith creation
- ✅ Wealthy noble smith creation
- ✅ Nomadic merchant creation
- ✅ Multi-dimensional variation testing

**Test Results:** All 21 tests passing ✅

---

## Configuration

### NPC Template Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "NPC Template",
  "type": "object",
  "properties": {
    "profession": { "type": "string" },
    "wealth_level": { "type": "integer", "minimum": 0, "maximum": 5 },
    "education": {
      "type": "string",
      "enum": ["NONE", "BASIC", "APPRENTICE", "SCHOLAR", "MASTER"]
    },
    "origin": {
      "type": "string",
      "enum": ["CITY", "VILLAGE", "RURAL", "NOMADIC", "FOREIGN"]
    },
    "experience": {
      "type": "string",
      "enum": ["NOVICE", "JOURNEYMAN", "VETERAN", "LEGENDARY"]
    },
    "personality_traits": {
      "type": "array",
      "items": { "type": "string" }
    },
    "inventory_quality": {
      "type": "string",
      "enum": ["COMMON", "UNCOMMON", "RARE", "EPIC", "LEGENDARY"]
    },
    "price_multiplier": { "type": "number", "minimum": 0.5, "maximum": 3.0 },
    "trade_items": {
      "type": "object",
      "additionalProperties": {
        "type": "number",
        "minimum": 0,
        "maximum": 1
      }
    }
  },
  "required": ["profession", "wealth_level"]
}
```

---

## Future Enhancements

1. **Dialog Tree Integration**: Use personality traits to modify dialog tone
2. **Quest Preference System**: Generate quest types based on education/experience
3. **Faction Affiliation**: Add faction dimension for political narratives
4. **Age Dimension**: Add age ranges for life experience variation
5. **Cultural Templates**: Expand origin types with cultural-specific narratives
6. **Dynamic Name Pools**: Load name pools from configuration files
7. **Backstory Expansion**: Add longer narrative arcs based on experience level
8. **Voice Modulation**: Map personality traits to voice pitch/speed for TTS integration

---

## API Reference

### NPCNarrativeDimensions

```java
public class NPCNarrativeDimensions {
    public NPCNarrativeDimensions setWealthLevel(int level)
    public NPCNarrativeDimensions setEducation(EducationLevel education)
    public NPCNarrativeDimensions setOrigin(OriginType origin)
    public NPCNarrativeDimensions setExperience(ExperienceLevel experience)
    public NPCNarrativeDimensions addTrait(PersonalityTrait trait)
    public NPCNarrativeDimensions removeTrait(PersonalityTrait trait)
    public boolean hasTrait(PersonalityTrait trait)
}
```

### NPCNarrativeGenerator

```java
public class NPCNarrativeGenerator {
    public NPCNarrativeGenerator(long seed)
    public String generateBackstory(NPCNarrativeDimensions dims, String profession, long npcSeed)
}
```

### NPCNameGenerator

```java
public class NPCNameGenerator {
    public NPCNameGenerator(long seed)
    public String generateName(int wealthLevel, OriginType origin, String profession, long npcSeed)
    public String generatePlainName(int wealthLevel, OriginType origin, String profession, long npcSeed)
}
```

---

## See Also

- [NPC Framework API Reference](../api-reference/npc-framework.md)
- [Quest Framework Integration](./quest-framework-integration.md)
- [World Generation Guide](../api/world-generation.md)
- [SF-06: NPC Framework Specification](../../../00-Argonath-Specifications/00-Architecture/SF-06-npc-framework.md)
