# Source_Files/GameWorld/dynamic_limits.cpp

## File Purpose
Manages configurable runtime limits for game entities (objects, monsters, projectiles, effects, pathfinding) in the Marathon engine. Provides XML-based configuration and automatic resizing of entity storage arrays when limits change.

## Core Responsibilities
- Maintain a static array of entity limits with reasonable defaults
- Parse XML `<dynamic_limits>` elements to override defaults per entity type
- Backup and restore original limit values across configuration reloads
- Coordinate array resizing for all entity lists (ObjectList, MonsterList, ProjectileList, EffectList, pathfinding)
- Expose limits to other modules via simple accessor function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_DynLimValueParser` | class | Parses individual `<objects>`, `<monsters>`, etc. elements; reads "value" attribute and validates range [0, 32767] |
| `XML_DynLimParser` | class | Root parser for `<dynamic_limits>` element; orchestrates resizing of all entity arrays |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dynamic_limits` | `uint16[NUMBER_OF_DYNAMIC_LIMITS]` | static | Current limits for 8 entity categories (objects, monsters, paths, projectiles, effects, rendered, local_collision, global_collision) |
| `original_dynamic_limits` | `uint16*` | static | Heap-allocated backup of defaults; allocated on first parse, used to restore on reset |
| `DynLimParser0ΓÇô7` | `XML_DynLimValueParser` | static | Eight pre-instantiated parsers, each bound to one element of `dynamic_limits[]` |
| `DynamicLimitsParser` | `XML_DynLimParser` | static | Root parser instance; children added dynamically in `DynamicLimits_GetParser()` |

## Key Functions / Methods

### XML_DynLimValueParser::Start
- **Signature:** `bool Start()`
- **Purpose:** Called when XML element begins parsing; allocates and populates backup of original limits on first invocation.
- **Inputs:** None (uses `original_dynamic_limits` and `dynamic_limits` globals)
- **Outputs/Return:** `true`
- **Side effects:** Allocates heap memory for `original_dynamic_limits` if `NULL`; sets `IsPresent = false`
- **Calls:** `malloc()`, `assert()`
- **Notes:** IdempotentΓÇösubsequent calls skip allocation because `original_dynamic_limits` is already set.

### XML_DynLimValueParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Process the "value" attribute, parse and validate it, update the bound limit.
- **Inputs:** `Tag` (attribute name), `Value` (string value)
- **Outputs/Return:** `true` if "value" tag was found and parsed successfully; `false` otherwise
- **Side effects:** Modifies `*ValuePtr` (the limit this parser is bound to); sets `IsPresent = true` on success
- **Calls:** `StringsEqual()`, `ReadBoundedUInt16Value()`
- **Notes:** Only "value" tag is recognized; calls `UnrecognizedTag()` on any other attribute.

### XML_DynLimValueParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Validate that required "value" attribute was present.
- **Inputs:** None
- **Outputs/Return:** `true` if `IsPresent` is set; `false` if value was missing
- **Side effects:** Calls `AttribsMissing()` if validation fails
- **Calls:** `AttribsMissing()`

### XML_DynLimParser::End
- **Signature:** `bool End()`
- **Purpose:** Finalize limit configuration by resizing all entity lists and pathfinding memory.
- **Inputs:** None (operates on current `dynamic_limits[]`)
- **Outputs/Return:** `true`
- **Side effects:** 
  - Resizes `EffectList`, `ObjectList`, `MonsterList`, `ProjectileList` using `MAXIMUM_*_PER_MAP` macros
  - Reallocates pathfinding memory via `allocate_pathfinding_memory()`
  - Note: dead code block (#if 0) suggests former `objlist_clear()` calls were removed
- **Calls:** `EffectList.resize()`, `ObjectList.resize()`, `MonsterList.resize()`, `ProjectileList.resize()`, `allocate_pathfinding_memory()`

### XML_DynLimParser::ResetValues
- **Signature:** `bool ResetValues()`
- **Purpose:** Restore limits to original defaults and reinitialize entity arrays.
- **Inputs:** None
- **Outputs/Return:** `true`
- **Side effects:** 
  - Restores `dynamic_limits[]` from `original_dynamic_limits[]`
  - Frees and nullifies `original_dynamic_limits`
  - Calls `End()` to resize arrays
- **Calls:** `End()`, `free()`
- **Notes:** Called when configuration is reset; allows reloading default limits.

### DynamicLimits_GetParser
- **Signature:** `XML_ElementParser *DynamicLimits_GetParser()`
- **Purpose:** Return and configure the root XML parser for dynamic limits.
- **Inputs:** None
- **Outputs/Return:** Pointer to `&DynamicLimitsParser`
- **Side effects:** Adds all eight `DynLimParser0ΓÇô7` as children of `DynamicLimitsParser` (on every call)
- **Calls:** `DynamicLimitsParser.AddChild()`
- **Notes:** Idempotent in effect (child addition is likely guarded or cumulative); called by XML loading machinery.

### get_dynamic_limit
- **Signature:** `uint16 get_dynamic_limit(int which)`
- **Purpose:** Query the current limit value for a given category.
- **Inputs:** `which` (enum value: `_dynamic_limit_objects`, `_dynamic_limit_monsters`, etc.)
- **Outputs/Return:** The current limit value (`dynamic_limits[which]`)
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple read-only accessor; used throughout the engine (e.g., `#define MAXIMUM_OBJECTS_PER_MAP (get_dynamic_limit(_dynamic_limit_objects))` in map.h).

## Control Flow Notes
This file operates in the **initialization/configuration phase**. The XML parser hierarchy is invoked during map/scenario loading to override default entity limits. When `End()` is called, entity arrays are resized, effectively preparing the world for the new limit values. The backup mechanism (`original_dynamic_limits`) supports configuration reload or reset scenarios.

## External Dependencies
- **Notable includes:**
  - `cseries.h` ΓÇö Cross-platform utilities, memory, strings
  - `dynamic_limits.h` ΓÇö Header with enum definitions and `get_dynamic_limit()` declaration
  - `map.h`, `effects.h`, `monsters.h`, `projectiles.h`, `flood_map.h` ΓÇö Forward declares or includes for entity list globals (`ObjectList`, `EffectList`, `MonsterList`, `ProjectileList`) and pathfinding
  - Standard `<string.h>`

- **External symbols used but not defined here:**
  - `StringsEqual()`, `ReadBoundedUInt16Value()`, `UnrecognizedTag()`, `AttribsMissing()` ΓÇö XML/string utilities (likely from CSeries)
  - `ObjectList`, `MonsterList`, `EffectList`, `ProjectileList` ΓÇö Entity storage vectors (defined in map/effects/monsters/projectiles modules)
  - `allocate_pathfinding_memory()` ΓÇö Pathfinding system (defined in flood_map module)
  - `XML_ElementParser` ΓÇö Base class for XML parsing (defined in XML_ElementParser.h)
