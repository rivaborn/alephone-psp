# Source_Files/GameWorld/map.cpp

## File Purpose
Core world management system for the game engine, handling map initialization, spatial entity management, and environmental systems. Manages the fundamental data structures that represent the game world (polygons, lines, objects) and facilitates core gameplay mechanics like collision detection, line-of-sight testing, and sound propagation.

## Core Responsibilities
- Allocate and initialize map memory structures for a new game/level
- Create and destroy dynamic game objects (scenery, items, garbage) in the world
- Maintain object-to-polygon spatial linkage for efficient queries
- Perform geometric queries: line-of-sight, point-in-polygon, polygon traversal
- Detect line crossings when entities traverse polygon boundaries
- Manage sound propagation including obstruction by walls and media
- Track and manage garbage object (corpse) limits per polygon/map
- Load/unload texture collections based on environment and level requirements
- Parse and configure texture environment mappings via XML
- Provide accessor functions for safe bounds-checked access to map entities

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `static_data` | struct | Static world metadata: environment, physics model, song index, level flags |
| `dynamic_data` | struct | Dynamic world state: tick count, random seed, entity counts, gameplay state |
| `object_data` | struct | Individual game object: position, polygon, shape, animation state, transfer mode |
| `polygon_data` | struct | Map polygon: vertices, height, texture, type (platform/trigger/etc), linked objects |
| `line_data` | struct | Map line/edge: endpoints, solidity flags, adjacent polygons, height info |
| `side_data` | struct | Wall surface: type, textures, control panel properties, transfer modes, lighting |
| `endpoint_data` | struct | Vertex with flags: solidity, elevation, adjacent heights, supporting polygon |
| `ambient_sound_image_data` | struct | Non-directional ambient sound tied to a polygon |
| `random_sound_image_data` | struct | Possibly-directional random ambient sound with pitch/volume variation |
| `XML_TextureEnvironmentParser` | class | XML parser for `<texture_env>` elements defining collection-per-environment mappings |
| `XML_TextureLoadingParser` | class | XML parser for `<texture_loading>` element controlling landscape loading |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `static_world` | `static_data*` | global | Current level's static metadata |
| `dynamic_world` | `dynamic_data*` | global | Current level's dynamic state |
| `ObjectList` | `vector<object_data>` | global | All game objects (max configurable) |
| `PolygonList` | `vector<polygon_data>` | global | All map polygons |
| `LineList` | `vector<line_data>` | global | All map lines |
| `SideList` | `vector<side_data>` | global | All map sides/walls |
| `EndpointList` | `vector<endpoint_data>` | global | All map vertices |
| `MonsterList` | `vector<monster_data>` | global | All monsters (imported from monsters.h) |
| `ProjectileList` | `vector<projectile_data>` | global | All projectiles (imported from projectiles.h) |
| `EffectList` | `vector<effect_data>` | global | All effects/explosions (imported from effects.h) |
| `PlatformList` | `vector<platform_data>` | global | All moving platforms |
| `AmbientSoundImageList` | `vector<ambient_sound_image_data>` | global | Ambient sounds per polygon |
| `RandomSoundImageList` | `vector<random_sound_image_data>` | global | Random ambient sounds per polygon |
| `MapIndexList` | `vector<int16>` | global | Sound source index lists |
| `AutomapLineList` | `vector<uint8>` | global | Bitfield: which lines are in automap |
| `AutomapPolygonList` | `vector<uint8>` | global | Bitfield: which polygons are in automap |
| `MapAnnotationList` | `vector<map_annotation>` | global | Map annotations (text labels) |
| `SavedObjectList` | `vector<map_object>` | global | Initial object placements from editor |
| `Environments` | `short[5][7]` | static | Texture collections per environment (5 envs, 7 collection slots each) |
| `map_collections` | `bool[NUMBER_OF_COLLECTIONS]` | static | Tracks which collections are currently needed |
| `media_effects` | `bool[NUMBER_OF_EFFECT_TYPES]` | static | Tracks which effect types are needed |
| `IntersectedObjects` | `vector<short>` | static | Reusable collision query buffer |
| `game_is_networked` | `bool` | global | Whether this is a networked game |
| `LandscapesLoaded` | `bool` | global | Whether Marathon 2/oo landscapes are active (vs. M1 mode) |
| `LoadedWallTexture` | `short` | global | First texture collection loaded (for infravision fog) |
| `OriginalEnvironments` | `short**` | static | Backup of original environment collections for XML reset |

## Key Functions / Methods

### allocate_map_memory
- Signature: `void allocate_map_memory(void)`
- Purpose: Initialize all world data structures at game start; allocate `static_world` and `dynamic_world`
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates dynamic memory for world structures; calls `allocate_player_memory()`
- Calls: `obj_clear()`, `allocate_player_memory()`
- Notes: Called once at engine startup; all the commented-out code shows the evolution from fixed arrays to STL vectors

### initialize_map_for_new_level
- Signature: `void initialize_map_for_new_level(void)`
- Purpose: Reset world state when entering a new level, preserving essential cross-level data
- Inputs: None
- Outputs/Return: None
- Side effects: Clears all dynamic entities (monsters, projectiles, effects) but preserves player count, tick count, random seed, game info, and civilian statistics
- Calls: `obj_clear()`, `objlist_clear()`, `Console::instance()->clear_saves()`
- Notes: Invoked on level load; must preserve cross-level state like tick counters for replay/network sync

### new_map_object3d
- Signature: `short new_map_object3d(world_point3d *location, short polygon_index, shape_descriptor shape, angle facing)`
- Purpose: Create a game object at a specific 3D location in a given polygon
- Inputs: 3D world location, containing polygon index, shape descriptor, facing angle
- Outputs/Return: Object index if created, `NONE` if object table full
- Side effects: Allocates object slot; inserts object into polygon's linked list (`polygon->first_object`); initializes object data
- Calls: `_new_map_object()`, `get_polygon_data()`
- Notes: The actual workhorse for object creation; called by `new_map_object2d()` and level loading code

### line_is_obstructed
- Signature: `bool line_is_obstructed(short polygon_index1, world_point2d *p1, short polygon_index2, world_point2d *p2)`
- Purpose: Test if a line segment between two points crosses solid map lines (walls); handles multi-polygon paths
- Inputs: Start polygon, start 2D point, end polygon, end 2D point
- Outputs/Return: `true` if the path is blocked by a solid line
- Side effects: None
- Calls: `_find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `LINE_IS_SOLID()`
- Notes: Core line-of-sight/obstruction test; iteratively traverses polygons, stopping if a solid line is hit or destination polygon unreachable; used by sound occlusion and visibility checks

### _find_line_crossed_leaving_polygon
- Signature: `short _find_line_crossed_leaving_polygon(short polygon_index, world_point2d *p0, world_point2d *p1, bool *last_line)`
- Purpose: Find which edge of a polygon is crossed when moving from p0 to p1
- Inputs: Current polygon, origin point, destination point
- Outputs/Return: Line index of the crossed edge, or `NONE` if destination is within polygon; `*last_line` set if p1 lies exactly on the edge
- Side effects: None
- Calls: `get_polygon_data()`, `get_endpoint_data()`
- Notes: Uses 2D cross-product math to test point-to-line orientation; the foundation for polygon-traversal algorithms

### mark_map_collections
- Signature: `void mark_map_collections(bool loading)`
- Purpose: Scan the current level's polygons and textures to determine which texture collections must be loaded/unloaded
- Inputs: `true` to mark for loading, `false` for unloading
- Outputs/Return: None
- Side effects: Sets `map_collections[collection_index]` based on textures used in polygons, sides, media, and scenery; calls `mark_collection_for_loading/unloading()` and `mark_effect_collections()`
- Calls: `get_side_data()`, `GET_DESCRIPTOR_COLLECTION()`, `get_media_data()`, `get_scenery_collection()`, `mark_collection_for_loading/unloading()`, `mark_effect_collections()`
- Notes: Critical for memory management; avoids loading unused texture collections

### reconnect_map_object_list
- Signature: `void reconnect_map_object_list(void)`
- Purpose: Rebuild the links between objects and their containing polygons after loading a level
- Inputs: None
- Outputs/Return: None
- Side effects: Rebuilds `polygon->first_object` and `object->next_object` chains from scratch
- Calls: `get_polygon_data()`
- Notes: Fixes the doubly-linked structure between objects and polygons; called after map load

### turn_object_to_shit
- Signature: `void turn_object_to_shit(short garbage_object_index)` (colorful naming from original Bungie code)
- Purpose: Mark an object as garbage (dead body) and enforce per-polygon and per-map garbage limits
- Inputs: Object index to mark as garbage
- Outputs/Return: None
- Side effects: Sets object owner to `_object_is_garbage`; removes oldest garbage object if limits exceeded; decrements `dynamic_world->garbage_object_count` as needed
- Calls: `get_object_data()`, `get_polygon_data()`, `remove_map_object()`, `GET_OBJECT_OWNER()`, `SET_OBJECT_OWNER()`
- Notes: Implements corpse limit logic respecting the `double_corpse_limit` preference; prevents performance degradation from corpse buildup

### point_is_monster_visible
- Signature: `bool point_is_monster_visible(short polygon_index, world_point2d *p, int32 *distance)`
- Purpose: Check if a point is visible to any nearby monsters and return the closest distance
- Inputs: Polygon containing the point, point location, output distance
- Outputs/Return: `true` if at least one monster can see the point; `*distance` set to closest monster distance
- Side effects: Uses static `IntersectedObjects` vector for collision query
- Calls: `possible_intersecting_monsters()`, `get_object_data()`, `guess_distance2d()`
- Notes: Used for monster activation; filters monsters in nearby polygons without explicit LOS testing (uses proximity heuristic)

### _sound_listener_proc
- Signature: `world_location3d *_sound_listener_proc(void)`
- Purpose: Return the current sound listener position (player camera location if game in progress)
- Inputs: None
- Outputs/Return: Pointer to listener location, or `NULL` if not in-game
- Side effects: None
- Calls: `get_game_state()`
- Notes: Callback used by SoundManager; provides reference point for all sound obstruction calculations

### _sound_obstructed_proc
- Signature: `uint16 _sound_obstructed_proc(world_location3d *source)`
- Purpose: Determine if sound from a source to the listener is obstructed, and if so, by what (walls, media, etc.)
- Inputs: Source location
- Outputs/Return: Flags: `_sound_was_obstructed`, `_sound_was_media_obstructed`, `_sound_was_media_muffled`
- Side effects: None
- Calls: `_sound_listener_proc()`, `line_is_obstructed()`, `get_polygon_data()`, `get_media_data()`
- Notes: Core audio obstruction system; media boundaries can muffle sound differently than solid walls

### handle_random_sound_image
- Signature: `void handle_random_sound_image(void)`
- Purpose: Update random ambient sound in the player's current polygon and play sound if its phase counter expires
- Inputs: None (uses `current_player`)
- Outputs/Return: None
- Side effects: Decrements `image->phase`; calls `DirectPlaySound()` when phase expires; applies volume/pitch/direction variance
- Calls: `get_polygon_data()`, `get_random_sound_image_data()`, `SoundManager::instance()->DirectPlaySound()`, `SoundManager::instance()->RandomSoundIndexToSoundIndex()`, `local_random()`
- Notes: Called once per game tick; implements periodic random ambient effects (wind, machinery hum, etc.)

### collection_in_environment
- Signature: `bool collection_in_environment(short collection_code, short environment_code)`
- Purpose: Check if a texture collection is part of a given environment's texture set
- Inputs: Collection index, environment code
- Outputs/Return: `true` if collection is in the environment's collection array
- Side effects: None
- Calls: `GET_COLLECTION()`
- Notes: Helper for texture environment validation

### mark_environment_collections
- Signature: `void mark_environment_collections(short environment_code, bool loading)`
- Purpose: Mark all texture collections for a given environment for loading or unloading
- Inputs: Environment code, loading flag
- Outputs/Return: None
- Side effects: Calls `mark_collection_for_loading/unloading()` for each collection in the environment; sets `LoadedWallTexture`; handles landscape loading based on `LandscapesLoaded`
- Calls: `mark_collection_for_loading/unloading()`
- Notes: Defers to XML-configured `Environments[environment_code][]` array for flexibility

### valid_point2d / valid_point3d
- Signature: `bool valid_point2d(world_point2d *p)` / `bool valid_point3d(world_point3d *p)`
- Purpose: Check if a point lies within the map's valid geometry
- Inputs: 2D or 3D point
- Outputs/Return: `true` if point is in a valid polygon with correct height
- Side effects: None
- Calls: `world_point_to_polygon_index()`, `get_polygon_data()`
- Notes: Used for validation before placing objects or testing gameplay triggers

### XML_TextureEnvironmentParser::AttributesDone
- Signature: `bool XML_TextureEnvironmentParser::AttributesDone()`
- Purpose: Finalize parsing of a `<texture_env>` element and update `Environments[index][which]`
- Inputs: None (uses member variables set by `HandleAttribute()`)
- Outputs/Return: `true` if all required attributes present and update succeeds
- Side effects: Modifies `Environments[Index][Which] = Coll`
- Calls: `ReadBoundedInt16Value()`
- Notes: XML parser callback; allows map creators to customize texture sets per environment without code changes

## Control Flow Notes

**Initialization sequence:** `allocate_map_memory()` ΓåÆ `initialize_map_for_new_game()` ΓåÆ level loading calls `initialize_map_for_new_level()` and populates entities

**Object lifecycle:** Entity creation via `new_map_object*()` ΓåÆ insertion into `polygon->first_object` linked list ΓåÆ updated each frame by monster/projectile/effect managers ΓåÆ removal via `remove_map_object()`

**Spatial queries:** Most queries (LOS, obstruction, visibility) follow a pattern of polygon traversal using `_find_line_crossed_leaving_polygon()` and multi-polygon checks

**Sound system:** Every frame, `handle_random_sound_image()` updates random ambient sounds; SoundManager calls `_sound_listener_proc()` and `_sound_obstructed_proc()` to compute 3D audio properties

**Texture loading:** Map/level designers configure collections per environment in XML; `mark_map_collections()` scans the level and requests loading only needed collections

## External Dependencies
- **map.h**: Structure definitions and constants for all map data types
- **cseries.h**: Basic utilities, macros (`obj_clear`, `vassert`), standard includes
- **interface.h**: Shape/collection management (`mark_collection_for_loading`, `get_media_data`)
- **monsters.h, player.h, projectiles.h, effects.h**: Game entity data structures and accessors
- **lightsource.h, media.h, scenery.h, platforms.h**: Feature-specific queries and management
- **SoundManager.h**: Audio playback and obstruction callbacks
- **lua_script.h**: Scripting support
- **Console.h**: Debugging/logging
- **XML_ElementParser.h**: XML parsing base class
- **std::vector, std::list, cstring, cstdlib, climits**: Standard library containers and utilities
