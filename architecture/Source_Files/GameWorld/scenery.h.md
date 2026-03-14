# Source_Files/GameWorld/scenery.h

## File Purpose
Header file declaring the public API for scenery object management in the Aleph One game engine. Provides functions for lifecycle (initialization, creation, destruction), animation, physics (damage), and query operations on static/interactive world objects. Includes XML configuration parsing support.

## Core Responsibilities
- Initialize and manage the scenery subsystem
- Create new scenery instances with world positioning
- Update scenery animation state each frame
- Apply damage and handle scenery destruction
- Query scenery metadata (dimensions, collection references)
- Support Lua scripting (add/delete/randomize scenery)
- Provide XML parser for scenery configuration loading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `object_location` | struct | Defines spatial position/orientation for scenery placement (defined elsewhere) |
| `world_distance` | typedef | Engine-native distance unit for radius/height queries (defined elsewhere) |
| `XML_ElementParser` | class | Base class for XML parsing hierarchy; used for scenery config deserialization |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_scenery
- Signature: `void initialize_scenery(void);`
- Purpose: Initializes the scenery system at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/resets scenery pools, preps animation state
- Calls: (Not visible; implementation in scenery.c)
- Notes: Called once at engine init; must precede all other scenery ops

### new_scenery
- Signature: `short new_scenery(struct object_location *location, short scenery_type);`
- Purpose: Instantiates a new scenery object at a world location
- Inputs: `location` (position/orientation), `scenery_type` (ID or enum)
- Outputs/Return: Object index (short) for future reference; likely -1 on failure
- Side effects: Allocates pool entry, updates world state
- Calls: (Not visible; likely calls dimension/collection getters internally)
- Notes: Returns handle for use with `damage_scenery()`, `deanimate_scenery()`

### animate_scenery
- Signature: `void animate_scenery(void);`
- Purpose: Advances animation state for all active scenery each frame
- Inputs: None
- Outputs/Return: None
- Side effects: Updates animation timers, sprite/frame indices in global scenery pool
- Calls: (Not visible; called per-frame in main loop)
- Notes: Performance-critical; likely iterates active pool only

### get_scenery_dimensions
- Signature: `void get_scenery_dimensions(short scenery_type, world_distance *radius, world_distance *height);`
- Purpose: Queries collision/render bounds for a scenery type
- Inputs: `scenery_type` (ID), pointers to output vars
- Outputs/Return: Writes `radius` and `height` via pointers
- Side effects: None (query-only)
- Calls: (Not visible; likely reads static type table)
- Notes: Used for collision detection and spatial queries

### damage_scenery
- Signature: `void damage_scenery(short object_index);`
- Purpose: Applies damage/destruction to a scenery object
- Inputs: `object_index` (from `new_scenery()`)
- Outputs/Return: None
- Side effects: May destroy object, trigger animation, update world state
- Calls: (Not visible; likely calls `get_damaged_scenery_collection()`)
- Notes: May be triggered by weapons, explosions, or physics

### get_scenery_collection / get_damaged_scenery_collection
- Signature: `bool get_scenery_collection(short scenery_type, short &collection);` and damaged variant
- Purpose: Maps scenery type to graphics collection ID for rendering
- Inputs: `scenery_type`
- Outputs/Return: Boolean success; writes `collection` reference via pointer
- Side effects: None (query-only)
- Calls: (Not visible; likely reads type ΓåÆ collection mapping table)
- Notes: Returns different collection for damaged vs. intact scenery

### Scenery_GetParser
- Signature: `XML_ElementParser *Scenery_GetParser();`
- Purpose: Returns the root XML parser for scenery configuration
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser hierarchy
- Side effects: None (query-only)
- Calls: (Not visible; returns pre-built parser tree)
- Notes: LP (Loren Petrich) addition for Marathon modding (XML map configs)

**Notes:**
- `deanimate_scenery()`, `randomize_scenery_shape()`, `randomize_scenery_shapes()` are helpers for Lua scripting; likely low-complexity wrappers.

## Control Flow Notes
**Initialization:** `initialize_scenery()` called once at engine startup.

**Per-frame update:** `animate_scenery()` likely called in the main game loop's update phase to advance animation state for all scenery.

**Event-driven:** `new_scenery()` called when maps load or Lua adds dynamic scenery; `damage_scenery()` called when projectiles/explosions hit scenery.

**Query:** `get_scenery_collection()` and `get_scenery_dimensions()` called during collision checks and rendering passes.

**Configuration loading:** `Scenery_GetParser()` used by XML map loader to parse scenery definitions from Marathon/Aleph One map files.

## External Dependencies
- **`XML_ElementParser.h`**: Provides XML parsing base class for scenery configuration deserialization (Loren Petrich's modding framework)
- **Types from elsewhere:** `object_location`, `world_distance`, `short` (int16) indices into global scenery pool
