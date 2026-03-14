# Source_Files/GameWorld/scenery.cpp

## File Purpose

Manages scenery objects in the game worldΓÇöstatic environmental objects that can be animated, destroyed, and configured. Provides creation, animation, collision, and damage handling for scenery, with XML-based definition support.

## Core Responsibilities

- Create and initialize scenery objects at map locations
- Manage animated scenery sequences and per-frame updates
- Handle scenery destruction and associated visual effects
- Query scenery properties (dimensions, texture collections)
- Parse and apply XML-based scenery definition overrides
- Maintain list of actively animated scenery for efficient frame updates

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `scenery_definition` | struct | Scenery type data: shape, destroyed_shape, flags, radius, height, destruction effect |
| `object_data` | struct | Runtime map object state (location, shape, animation sequence, owner) |
| `XML_SceneryShapesParser` | class | XML element parser for shape descriptors (normal/destroyed variants) |
| `XML_SceneryObjectParser` | class | XML element parser for scenery object attribute definitions |
| `shape_descriptor` | typedef | Packed collection/shape index identifier |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AnimatedSceneryObjects` | `vector<short>` | static | Indices of scenery objects requiring per-frame animation |
| `SceneryNormalParser` | `XML_SceneryShapesParser` | static | XML parser for undamaged scenery shapes |
| `SceneryDestroyedParser` | `XML_SceneryShapesParser` | static | XML parser for destroyed scenery shapes |
| `SceneryObjectParser` | `XML_SceneryObjectParser` | static | XML parser for scenery object attributes |
| `SceneryParser` | `XML_ElementParser` | static | Root XML element parser for scenery definitions |
| `original_scenery_definitions` | `scenery_definition*` | static | Backup of original definitions before XML overrides |

## Key Functions / Methods

### initialize_scenery
- **Signature:** `void initialize_scenery(void)`
- **Purpose:** Initialize scenery system and preallocate space for animated objects
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reserves 32 slots in `AnimatedSceneryObjects` vector
- **Calls:** `std::vector::reserve()`
- **Notes:** Called once at game initialization

### new_scenery
- **Signature:** `short new_scenery(struct object_location *location, short scenery_type)`
- **Purpose:** Create and place a new scenery object in the world
- **Inputs:** `location` (world position/polygon), `scenery_type` (definition index)
- **Outputs/Return:** Object index on success; `NONE` if definition invalid or map object creation fails
- **Side effects:** Creates map object, sets owner to `_object_is_scenery`, sets solidity from definition flags, stores scenery_type in `object->permutation`
- **Calls:** `get_scenery_definition()`, `new_map_object()`, `get_object_data()`, `SET_OBJECT_OWNER()`, `SET_OBJECT_SOLIDITY()`
- **Notes:** Bounds checks scenery_type via `get_scenery_definition()`

### animate_scenery
- **Signature:** `void animate_scenery(void)`
- **Purpose:** Advance animation frame for all active animated scenery objects
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies animation sequence and frame state for each object in `AnimatedSceneryObjects`
- **Calls:** `animate_object()` for each index
- **Notes:** Called once per game tick; no-op if list is empty

### deanimate_scenery
- **Signature:** `void deanimate_scenery(short object_index)`
- **Purpose:** Remove a scenery object from the animated list
- **Inputs:** `object_index` to remove
- **Outputs/Return:** None
- **Side effects:** Erases matching entry from `AnimatedSceneryObjects`
- **Calls:** `std::vector::erase()`
- **Notes:** Safe if object not in list; uses linear search

### randomize_scenery_shape
- **Signature:** `void randomize_scenery_shape(short object_index)`
- **Purpose:** Initialize animation sequence for a single scenery object
- **Inputs:** `object_index`
- **Outputs/Return:** None
- **Side effects:** May add object to `AnimatedSceneryObjects` if sequence created
- **Calls:** `get_object_data()`, `get_scenery_definition()`, `randomize_object_sequence()`
- **Notes:** Adds to animated list only if `randomize_object_sequence()` returns false

### randomize_scenery_shapes
- **Signature:** `void randomize_scenery_shapes(void)`
- **Purpose:** Reinitialize all scenery objects with randomized animation sequences
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears and repopulates `AnimatedSceneryObjects`; iterates all map objects
- **Calls:** `get_object_data()`, `get_scenery_definition()`, `randomize_object_sequence()`, `GET_OBJECT_OWNER()`
- **Notes:** O(MAXIMUM_OBJECTS_PER_MAP) complexity; typically called at level load

### get_scenery_dimensions
- **Signature:** `void get_scenery_dimensions(short scenery_type, world_distance *radius, world_distance *height)`
- **Purpose:** Retrieve collision bounding box dimensions for a scenery type
- **Inputs:** `scenery_type`, output pointers
- **Outputs/Return:** Sets `*radius` and `*height`
- **Side effects:** None
- **Calls:** `get_scenery_definition()`
- **Notes:** Sets both to 0 if definition not found (fallback); used for pathfinding/collision

### damage_scenery
- **Signature:** `void damage_scenery(short object_index)`
- **Purpose:** Handle destruction of a scenery object when damaged
- **Inputs:** `object_index`
- **Outputs/Return:** None
- **Side effects:** Swaps shape to `destroyed_shape`, creates destruction effect, changes owner to `_object_is_normal`
- **Calls:** `get_object_data()`, `get_scenery_definition()`, `new_effect()`
- **Notes:** Only destructible if `_scenery_can_be_destroyed` flag set; skips effect if `destroyed_effect == NONE`

### get_scenery_collection
- **Signature:** `bool get_scenery_collection(short scenery_type, short& collection)`
- **Purpose:** Extract texture collection index from scenery type
- **Inputs:** `scenery_type`
- **Outputs/Return:** `true` if found; sets `collection` reference
- **Side effects:** None
- **Calls:** `get_scenery_definition()`, `GET_DESCRIPTOR_COLLECTION()` macro
- **Notes:** Used for mark/load collection operations

### get_damaged_scenery_collection
- **Signature:** `bool get_damaged_scenery_collection(short scenery_type, short& collection)`
- **Purpose:** Extract destroyed variant's texture collection index
- **Inputs:** `scenery_type`
- **Outputs/Return:** `true` if scenery is destructible and collection obtained
- **Side effects:** None
- **Calls:** `get_scenery_definition()`, `GET_DESCRIPTOR_COLLECTION()`
- **Notes:** Returns `false` if scenery lacks `_scenery_can_be_destroyed` flag

### Scenery_GetParser
- **Signature:** `XML_ElementParser *Scenery_GetParser()`
- **Purpose:** Build and return XML parser hierarchy for scenery definition documents
- **Inputs:** None
- **Outputs/Return:** Pointer to root `SceneryParser` element
- **Side effects:** Chains parsers via `AddChild()` calls; runs each time called (repeated setup acceptable)
- **Calls:** `AddChild()` to attach shape parsers and object parser
- **Notes:** Parser tree: SceneryParser ΓåÆ SceneryObjectParser ΓåÆ (SceneryNormalParser, SceneryDestroyedParser) ΓåÆ Shape_GetParser()

## Control Flow Notes

**Initialization ΓåÆ Randomization ΓåÆ Per-Frame Update ΓåÆ Damage:**

1. `initialize_scenery()` reserves vector space (one-time startup)
2. Map load calls `randomize_scenery_shapes()` to populate `AnimatedSceneryObjects` and set up animation frames
3. Each game tick, `animate_scenery()` advances frames for active objects
4. Damage events trigger `damage_scenery()`, which changes shape and transitions object state
5. XML parser (`Scenery_GetParser()`) can override definitions before level starts

The `AnimatedSceneryObjects` vector is the core runtime state; objects only appear here if animation frames exist. The `scenery_definitions` array (from header) is the static blueprint for all scenery types.

## External Dependencies

- **cseries.h** ΓÇö base types, macros (`NONE`, `TEST_FLAG`), assertions
- **map.h** ΓÇö `object_data`, `object_location`, `new_map_object()`, `get_object_data()`, `animate_object()`, `randomize_object_sequence()`, object owner/solidity flag macros
- **effects.h** ΓÇö `new_effect()`, effect type constants
- **ShapesParser.h** ΓÇö `Shape_GetParser()`, `Shape_SetPointer()`, shape descriptor utilities
- **scenery_definitions.h** ΓÇö `scenery_definitions[]` array, `NUMBER_OF_SCENERY_DEFINITIONS`
- Included but not visibly used: render.h, interface.h, flood_map.h, monsters.h, projectiles.h, player.h, platforms.h (likely for compilation dependencies)

**Utility function (defined elsewhere):**
- `GetMemberWithBounds()` ΓÇö bounds-checked array access with NULL return on out-of-range
