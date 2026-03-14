# Source_Files/GameWorld/devices.cpp
## File Purpose
Manages interactive control panels and device switches in the game worldΓÇödoors, refuel stations, save points, computer terminals, and various trigger switches. Handles player interaction, panel state management, texture updates, and networked save coordination.

## Core Responsibilities
- Define and initialize control panel types with audio/visual properties
- Detect and handle player action-key activation of nearby panels
- Update continuous interactions (oxygen/shield recharge, auto-save polling)
- Toggle panel states and cascade effects to linked lights/platforms
- Validate switch accessibility (lighting, item requirements, projectile-only)
- Play panel sounds and update textures based on state
- Coordinate pattern-buffer saves in networked games
- Parse and apply XML configuration overrides for panel properties

## External Dependencies
- **map.h**: Side, line, polygon data; map geometry queries
- **player.h**: Player state, items, control panel side index
- **monsters.h**: Monster dimensions and visibility  
- **platforms.h**: Platform state modification  
- **SoundManager.h**: Sound playback API  
- **computer_interface.h**: Terminal entry point  
- **lightsource.h** (implicit): Light status queries  
- **lua_script.h**: Script hook functions (`L_Call_*`)
- **Defined elsewhere**: `dynamic_world`, `map_sides`, `players`, `local_player`, save/load functions, light/platform control functions, XML parser base class

# Source_Files/GameWorld/dynamic_limits.cpp
## File Purpose
Manages configurable runtime limits for game entities (objects, monsters, projectiles, effects, pathfinding) in the Marathon engine. Provides XML-based configuration and automatic resizing of entity storage arrays when limits change.

## Core Responsibilities
- Maintain a static array of entity limits with reasonable defaults
- Parse XML `<dynamic_limits>` elements to override defaults per entity type
- Backup and restore original limit values across configuration reloads
- Coordinate array resizing for all entity lists (ObjectList, MonsterList, ProjectileList, EffectList, pathfinding)
- Expose limits to other modules via simple accessor function

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

# Source_Files/GameWorld/dynamic_limits.h
## File Purpose
Defines the interface for managing dynamically-configured resource limits in the game engine. This header exposes functions to query limits on game entities (objects, NPCs, projectiles, effects, etc.) whose values are initialized from XML configuration files rather than compile-time constants.

## Core Responsibilities
- Define enumeration of all limit types (objects, monsters, projectiles, effects, collision buffers, etc.)
- Provide XML parser integration for loading limit configurations
- Expose accessor interface to retrieve current limits at runtime

## External Dependencies
- `XML_ElementParser.h` ΓÇô base parser class for XML configuration system
- Standard C headers (implicit: `<stdio.h>` in bundled XML_ElementParser.h)
- Implementation file (not provided) likely includes engine-specific types and storage

# Source_Files/GameWorld/editor.h
## File Purpose
Header file defining editor-related constants, version macros, and data structures for the Marathon map editor. Specifies version compatibility across Marathon editions and establishes bounds for map geometry and patrol paths.

## Core Responsibilities
- Define data version constants for Marathon One, Two, and Infinity editions
- Specify valid coordinate and height ranges for map geometry
- Define data structure for map metadata storage
- Define data structure for guard patrol path control points and flags
- Establish compile-time constraints on map complexity

## External Dependencies
- `SHORT_MIN`, `SHORT_MAX` ΓÇö bounds constants (inferred from standard library or platform headers)
- `WORLD_ONE` ΓÇö world unit scale constant (defined elsewhere)
- `LEVEL_NAME_LENGTH` ΓÇö string buffer size (defined elsewhere)
- `world_point2d` ΓÇö 2D coordinate type (defined elsewhere)

**Notes:**
- Version constants (`MARATHON_ONE_DATA_VERSION` = 0, 1, 2) suggest multi-version format compatibility.
- Height bounds allow ┬▒8 world units with 1-unit minimum ceiling clearance (`MINIMUM_CEILING_HEIGHT`).
- `INVALID_HEIGHT` sentinel value below minimum floor for validation.
- Guard path struct supports up to 20 control points per path with per-point polygon association.
- `MAX_LINES_PER_VERTEX` (15) constrains polygon complexity for editor validation.

# Source_Files/GameWorld/effect_definitions.h
## File Purpose
Defines effect metadata structures and a table of effect definitions for the game. Effects are visual/audio animations triggered by game events (projectile impacts, entity deaths, splashes, etc.). This header centralizes effect configuration for the Aleph One game engine (Marathon-style).

## Core Responsibilities
- Define effect behavior flags (`_end_when_animation_loops`, `_sound_only`, `_media_effect`, etc.)
- Define the `effect_definition` struct encapsulating effect metadata
- Declare a mutable static array of effect definitions
- Provide a const table of ~60 predefined effect configurations mapped to game events
- Declare serialization functions for effect definitions (pack/unpack from streams)

## External Dependencies
- **Collection macros** (`_collection_rocket`, `_collection_fighter`, `_collection_compiler`, etc.): define sprite/animation libraries
- **Frequency constants** (`_normal_frequency`, `_higher_frequency`, `_lower_frequency`): sound pitch values
- **BUILD_COLLECTION()** macro: constructs collection identifiers
- **Sound enum** (`_snd_teleport_in`): sound effect IDs
- **Constants** (`NUMBER_OF_EFFECT_TYPES`, `TICKS_PER_SECOND`, `NONE`): indices and time values
- All defined elsewhere in codebase

---

**Note:** Line 2 contains a typo in the include guard (`__EFFECT_DEFINTIIONS_H` should be `__EFFECT_DEFINITIONS_H`), but the actual guard on line 1 is spelled correctly, so the header will include properly.

# Source_Files/GameWorld/effects.cpp
## File Purpose
Manages temporary visual and audio effects in the game worldΓÇöexplosions, teleport effects, blood splashes, sparks, and other short-lived phenomena. Effects are tied to animated map objects and are automatically removed when their animations complete or after a delay period expires.

## Core Responsibilities
- Create new effects at world locations with specified types and animations
- Update all active effects each frame, advancing animations and delay timers
- Remove effects when animations terminate or all nonpersistent effects are cleared
- Manage effect-definition lookups and validation
- Handle specialized teleport effects (fold in/fold out)
- Mark effect collections for loading/unloading as needed
- Serialize and deserialize effect data for save games and network play

## External Dependencies
- **Notable includes**: `cseries.h` (common types/macros), `map.h` (world structures, object accessors), `interface.h` (shape/sound data), `effects.h` (effect declarations), `SoundManager.h` (sound playback), `Packing.h` (serialization macros), `effect_definitions.h` (hardcoded effect table).
- **Defined elsewhere**: `effects` array (EffectList global in map.cpp), `effect_definitions` template (effect_definitions.h), `new_map_object3d()`, `animate_object()`, `play_object_sound()`, `remove_map_object()`, `get_shape_animation_data()` (map/objects/shapes modules).

# Source_Files/GameWorld/effects.h
## File Purpose
Header file for managing visual effects (explosions, blood splashes, water effects, etc.) in the Marathon/Aleph One game engine. Defines effect data structures, enumeration of all effect types, and function prototypes for creating, updating, and serializing effects.

## Core Responsibilities
- Define the `effect_data` structure to store active effect instances
- Enumerate all 80+ effect types (explosions, ricochets, splashes, sparks, detonations)
- Maintain a dynamic array (`EffectList`) of active effects in the current game world
- Provide creation and lifecycle management (create, update, remove effects)
- Support serialization/deserialization of effect data and definitions
- Retrieve and manage effect definitions (loaded from resources)

## External Dependencies
- `dynamic_limits.h`: Provides `get_dynamic_limit()` to retrieve dynamic effect cap
- External types: `world_point3d`, `angle` (defined elsewhere)
- C++ STL: `vector<>` for effect list management
- Slot management macros: `SLOT_IS_USED()`, `SLOT_IS_FREE()`, `MARK_SLOT_AS_FREE()`, `MARK_SLOT_AS_USED()` (defined elsewhere)

# Source_Files/GameWorld/flood_map.cpp
## File Purpose
Implements a graph-traversal/flood-fill algorithm for pathfinding over a polygon-based game world map. Supports multiple search strategies (breadth-first, best-first) with iterative node expansion and path reconstruction via backtracking.

## Core Responsibilities
- Allocate and manage memory for the search node pool and visited-polygon tracker
- Execute one iteration of the selected search algorithm, returning the next polygon to expand
- Reconstruct paths by walking backward through the parent node chain
- Support cost-based node evaluation (best-first search) and optional caller-provided flags
- Select random nodes with optional directional bias for AI pathfinding

## External Dependencies
- **Includes**: `cseries.h` (core utilities, assert), `map.h` (polygon_data, world types), `flood_map.h` (public interface)
- **Defined elsewhere**: `get_polygon_data()`, `objlist_set()`, `find_center_of_polygon()`, `global_random()`, `dynamic_world`
- **Types**: `world_vector2d`, `world_point2d`, `polygon_data`, `cost_proc_ptr` (function pointer)

# Source_Files/GameWorld/flood_map.h
## File Purpose
Defines the flood-map pathfinding algorithm interface and memory management for the game world. Flood-map is a spatial search technique that finds paths through the game's polygon-based terrain by exploring connected polygons in various traversal orders.

## Core Responsibilities
- Define flood-map search modes (depth-first, breadth-first, flagged, best-first)
- Declare cost function pointer types for custom path weighting
- Manage path creation, traversal, and deletion
- Allocate and manage memory for pathfinding structures
- Perform reverse flood-map queries and depth calculations
- Support random node selection for AI navigation

## External Dependencies
- `world_point2d`, `world_vector2d`, `world_distance` types (defined elsewhere; world geometry types)
- C standard types: `short`, `long`, `bool`, `void`

# Source_Files/GameWorld/item_definitions.h
## File Purpose
Defines the data structure and lookup table for all in-game items (weapons, ammunition, powerups, keys, game balls). Provides metadata mapping for item behavior, display names, visual shapes, and spawn limits. Part of the Aleph One/Marathon engine's item system.

## Core Responsibilities
- Define `item_definition` struct to describe item properties
- Provide static initialization table for all 40+ item types in the game
- Map item kinds to singular/plural name IDs for localization
- Associate items with their shape descriptors for rendering
- Enforce per-player item limits (e.g., max 2 pistols, max 1 fusion pistol)
- Flag items with environment restrictions (vacuum-dangerous weapons)

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()` (defined elsewhereΓÇöshape builder macros)
- **Enums** (defined elsewhere):
  - Item kinds: `_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball`
  - Special values: `UNONE`, `NONE`
  - Environments: `_environment_vacuum`, `_environment_single_player`
- **Type**: `shape_descriptor` (defined elsewhereΓÇölikely a collection/shape reference)

**Notes**: 
- The conditional guard `#ifndef DONT_REPEAT_DEFINITIONS` suggests this table is also mirrored in `script_instructions.cpp` (Pfhortran scripting).
- Items without visual representation use `UNONE` for shape.
- Powerups use `NONE` for both singular and plural name IDs (likely hardcoded visuals with no text labels).
- Vacuum-restricted items cannot spawn in airless environments.

# Source_Files/GameWorld/items.cpp
## File Purpose
Manages item lifecycle in the game world including spawning, pickup mechanics, inventory tracking, animation, and XML-based configuration of item properties. Handles both automatic and manual item collection, network-safe item placement, and environment-specific item availability.

## Core Responsibilities
- Create and spawn items on the map with network/difficulty context
- Implement item pickup logic with special handling for powerups, balls, weapons, and ammo
- Provide automatic nearby-item collection with geometry validation
- Trigger hidden items via zone-based flood-fill activation
- Update item animations and handle randomization of non-animated items
- Manage item definitions with runtime XML reconfiguration
- Track player inventory and item availability per environment/gamemode
- Lua scripting integration for item lifecycle events

## External Dependencies
- **map.h:** `object_data`, `object_location`, `polygon_data`, `world_point3d`, `world_point2d`, `new_map_object()`, `remove_map_object()`, `get_object_data()`, `get_polygon_data()`, `find_line_crossed_leaving_polygon()`, `find_adjacent_polygon()`, `get_line_data()`, `teleport_object_in()`, `randomize_object_sequence()`, `animate_object()`
- **player.h:** `player_data`, `get_player_data()`, `process_new_item_for_reloading()`, `mark_player_inventory_as_dirty()`, `legal_player_powerup()`, `process_player_powerup()`
- **monsters.h:** `get_monster_data()`, `get_monster_dimensions()`
- **interface.h:** `get_shape_animation_data()`, `get_item_kind()` (defined elsewhere)
- **SoundManager.h:** `SoundManager::instance()->PlayLocalSound()`, sound index accessors
- **platforms.h:** `get_platform_data()`, `PLATFORM_IS_MOVING()`
- **fades.h:** `start_fade()`
- **lua_script.h:** `L_Call_Item_Created()`, `L_Call_Got_Item()`
- **network_games.h:** `current_game_has_balls()`, `dynamic_world->player_count`, game type/environment flags
- **flood_map.h:** `flood_map()`
- **item_definitions.h:** `item_definitions[]` array, `NUMBER_OF_DEFINED_ITEMS`

# Source_Files/GameWorld/items.h
## File Purpose
Header file defining the item system for the game world. Declares item type enumerations, item management functions, and XML configuration support. Items include weapons, ammunition, powerups, and interactive balls used in multiplayer modes.

## Core Responsibilities
- Define item class types (weapons, ammunition, powerups, balls, etc.) and specific item IDs
- Create and place items in the world (`new_item`, `new_item_in_random_location`)
- Manage player inventory and item pickup logic (`try_and_add_player_item`, `swipe_nearby_items`)
- Query item metadata and properties (`get_item_kind`, `get_item_shape`, `get_item_definition_external`)
- Validate items in current environment and count inventory capacity
- Handle item interaction triggers (`trigger_nearby_items`)
- Track ball ownership for multiplayer game modes (`find_player_ball_color`)
- Provide frame-based animation and initialization via callbacks
- Support XML-driven configuration of item properties

## External Dependencies
- **XML_ElementParser.h**: C++ XML parsing framework for configuration
- **object_location**: Structure defined elsewhere (world position/polygon)
- **item_definition**: Structure defined elsewhere (item metadata, damage, animation, etc.)
- Callers in: game_window.c, scenery system, player/inventory systems, collision handlers

# Source_Files/GameWorld/lightsource.cpp
## File Purpose
Implements dynamic light source management for the Aleph One game engine (Marathon series). Manages creation, animation, and state transitions of in-game lights with support for multiple lighting types and animation functions. Handles both runtime light updates and serialization/deserialization of light data.

## Core Responsibilities
- Create and manage light instances in `LightList` vector
- Update light intensity each frame based on lighting function and animation phase
- Handle light state transitions (becoming active/inactive, primary/secondary states)
- Manage stateless (six-phase) lights vs. state-machine lights
- Query and modify light status (on/off) and intensity values
- Pack/unpack light data structures for save/load and network synchronization
- Convert legacy Marathon 1 light format to new format
- Hook into Lua scripting system for light activation events
- Support per-tag light control (toggle multiple lights by tag value)

## External Dependencies
- **Includes**: `cseries.h` (common definitions), `map.h` (map structure and constants), `lightsource.h` (header), `Packing.h` (serialization), `lua_script.h` (scripting hooks)
- **Symbols defined elsewhere**: 
  - `MAXIMUM_LIGHTS_PER_MAP`, `LIGHT_IS_INITIALLY_ACTIVE()`, `LIGHT_IS_STATELESS()` macros from lightsource.h
  - `GetMemberWithBounds()`, `SLOT_IS_USED()`, `MARK_SLOT_AS_USED()` macros (likely from map.h)
  - `global_random()` (random number generator)
  - `cosine_table[]`, `HALF_CIRCLE`, `TRIG_MAGNITUDE`, `TRIG_SHIFT` (trig lookup table)
  - `L_Call_Light_Activated()` (Lua scripting interface)
  - `assume_correct_switch_position()` (panel/switch synchronization)
  - `vhalt()`, `csprintf()`, `temporary` (error handling/formatting)
  - `StreamToValue()`, `ValueToStream()` functions from Packing.h

# Source_Files/GameWorld/lightsource.h
## File Purpose
Header defining the lighting system for the game engine. Declares light data structures, state management, and animation parameters, supporting both modern and legacy (Marathon 1) light formats for backward compatibility.

## Core Responsibilities
- Define static (template) and dynamic (runtime) light data structures
- Specify light types (normal, strobe, media) and state transitions
- Define lighting transition functions (constant, linear, smooth, flicker)
- Provide flag macros for light configuration
- Declare light management, query, and frame-update functions
- Support serialization/deserialization of light data
- Maintain backward compatibility with Marathon 1 map format

## External Dependencies
- `#include <vector>` ΓÇô STL vector for LightList
- Custom types: `_fixed` (fixed-point), `int16`, `uint16`, `uint8`, `size_t`
- Macro utilities: `TEST_FLAG16`, `SET_FLAG16` (defined elsewhere)

**Notes**: Serialization helpers (`pack_old_light_data`, `unpack_static_light_data`, etc.) handle map I/O; `get_defaults_for_light_type` provides factory configs.

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

# Source_Files/GameWorld/map.h
## File Purpose
Central header for Marathon's map/world system. Defines all data structures for game geometry (polygons, lines, endpoints), runtime objects (monsters, items, effects), game state management, and provides the primary API for map initialization, object placement, collision queries, and in-game events (damage, device activation, light changes).

## Core Responsibilities
- Define map geometry primitives: endpoints, lines, sides, polygons with per-entity properties (height, texture, flags)
- Define runtime object representation and lifecycle management (creation, deletion, translation)
- Declare game state structs: `game_data`, `dynamic_data`, `static_data` tracking ticks, entity counts, difficulty, mission state
- Provide map traversal queries: polygon lookup from coordinates, adjacency, line crossing detection, distance calculations
- Support serialization: pack/unpack functions for all major data types for save/load
- Declare frame update entry point (`update_world()`) and event handlers (polygon damage, device state changes, light updates)
- Manage sound placement: ambient and random sound image data structures and playback API
- Support difficulty-aware spawning: object frequency definitions, random placement

## External Dependencies

- **Includes:**
  - `csmacros.h` ΓÇô flag manipulation macros (TEST_FLAG, SET_FLAG), numeric utilities (MIN, MAX, ABS)
  - `world.h` ΓÇô coordinate types (`world_point2d`, `world_point3d`, `world_distance`, `angle`), trigonometry
  - `dynamic_limits.h` ΓÇô dynamic entity count accessor (`get_dynamic_limit()`)
  - `XML_ElementParser.h` ΓÇô XML parsing infrastructure (texture-loading parser)
  - `shape_descriptors.h` ΓÇô shape/collection enumeration and macros
  - `<vector>` ΓÇô STL dynamic arrays (replaces old fixed arrays)

- **Defined elsewhere (assumed from context):**
  - `_fixed` ΓÇô fixed-point arithmetic type (likely 16.16 format)
  - `uint8`, `int16`, `uint16`, `int32`, `uint32` ΓÇô integer types (from cstypes.h, included indirectly)
  - `XML_ElementParser` ΓÇô base class for XML element parsers
  - Game constants: `NONE` (sentinel index), `WORLD_ONE` (unit scale), `FIXED_ONE`

# Source_Files/GameWorld/map_constructors.cpp
## File Purpose
Constructs and initializes map geometry elements (polygons, lines, sides, endpoints) during map loading and editing. Recalculates derived geometric properties and precalculates collision/neighbor data. Provides binary serialization (pack/unpack) for saving and loading map state.

## Core Responsibilities
- **Geometry creation**: Create new sides, endpoints, lines, polygons with proper ownership and linkage
- **Data recalculation**: Recompute derived properties (area, endpoints, adjacent polygons, heights, lightsources)
- **Redundancy elimination**: Calculate values that can be inferred from geometry but are cached for performance
- **Collision preprocessing**: Build exclusion zones and neighbor lists via flood-fill for runtime impassability checks
- **Serialization**: Pack/unpack all map data types to/from byte streams (big-endian format)
- **Lightsource assignment**: Infer which light sources illuminate polygon sides based on geometry type

## External Dependencies
- **Includes**: `cseries.h` (basic types, macros), `map.h` (data structures), `flood_map.h` (flood_map function), `Packing.h` (serialization macros)
- **Global arrays** (defined elsewhere): `SideList`, `LineList`, `EndpointList`, `PolygonList`, `MapIndexList` (all std::vector<>)
- **Global pointers**: `static_world` (map metadata), `dynamic_world` (runtime counts and state)
- **Utility functions** (defined elsewhere): `get_side_data()`, `get_line_data()`, `get_polygon_data()`, `get_endpoint_data()` (accessors), `find_center_of_polygon()`, `flood_map()`, `clockwise_endpoint_in_line()`, `find_adjacent_polygon()`, `push_out_line()`, `distance2d()`, `obj_clear()`, `find_center_of_polygon()`
- **Macros from map.h**: `POLYGON_IS_DETACHED()`, `LINE_IS_SOLID()`, `LINE_IS_TRANSPARENT()`, `LINE_IS_ELEVATION()`, `SET_*` flag operations

# Source_Files/GameWorld/marathon2.cpp
## File Purpose
Core game world initialization and main update loop for the Marathon engine. Orchestrates all per-tick game state updates (player movement, monsters, projectiles, effects, platforms, etc.), manages level transitions, and implements client-side prediction for network multiplayer.

## Core Responsibilities
- Game world initialization (memory allocation, subsystem setup)
- Main game loop (`update_world()`) coordinating all entity updates
- Action queue management for input/Lua/prediction
- Level entry/exit and cleanup
- Game-state saving/restoration for client prediction
- Polygon trigger handling (monster activation, platform control, damage zones)
- Mission objective completion evaluation
- Damage calculation and difficulty scaling

## External Dependencies
- **Game subsystems** (defined elsewhere): `map.h`, `render.h`, `interface.h`, `flood_map.h`, `effects.h`, `monsters.h`, `projectiles.h`, `player.h`, `network.h`, `scenery.h`, `platforms.h`, `lightsource.h`, `media.h`, `items.h`, `weapons.h`.
- **Managers**: `Music.h`, `SoundManager.h`, `game_window.h`, `network_games.h`, `tags.h`, `AnimatedTextures.h`, `ChaseCam.h`, `OGL_Setup.h`.
- **Scripting & I/O**: `lua_script.h`, `ActionQueues.h`, `Logging.h`, `Console.h`, `screen.h`, `shell.h`.
- **Utilities**: `cseries.h` (standard types and macros).

**Key external symbols** (defined elsewhere):
- `dynamic_world`, `static_world` ΓÇö global game state pointers.
- `GetRealActionQueues()`, `GetLuaActionQueues()` ΓÇö input source accessors.
- `map_polygons[]` ΓÇö polygon array (from map.h).
- Entity data accessors: `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_polygon_data()`.
- Subsystem update functions: `update_players()`, `move_monsters()`, `update_effects()`, etc.
- Network functions: `NetProcessMessagesInGame()`, `NetSync()`, `game_is_networked`, `NetGetNetTime()`.

# Source_Files/GameWorld/media.cpp
## File Purpose

Manages in-game media (liquids/fluids) such as water, lava, goo, sewage, and Jjaro. Handles instantiation, per-frame updates, querying media properties (damage, sounds, visual effects), and serialization/XML loading of media definitions.

## Core Responsibilities

- Creating and destroying media instances in the game world
- Updating media height, texture, and flow direction each frame
- Querying media properties: damage, sounds, detonation effects, fade effects
- Checking media presence in specific environment codes
- Serializing/deserializing media data to byte streams
- Parsing and applying XML-based media definition overrides
- Counting active media slots for save-game compatibility

## External Dependencies

- **map.h** ΓÇö `media_data` struct, `SLOT_IS_USED/MARK_SLOT_AS_USED` macros, slot allocation conventions
- **effects.h** ΓÇö `NUMBER_OF_EFFECT_TYPES` enum
- **fades.h** ΓÇö Fade effect types (e.g., `_fade_tint_blue`)
- **lightsource.h** ΓÇö `get_light_intensity()` function
- **SoundManager.h** ΓÇö Sound IDs and definitions
- **DamageParser.h** ΓÇö Damage parsing utilities
- **Packing.h** ΓÇö Serialization macros (`StreamToValue`, `ValueToStream`)
- **media_definitions.h** ΓÇö `media_definitions[]` array and `NUMBER_OF_MEDIA_TYPES` (included from header, defined elsewhere)

**Defined elsewhere:**
- `media_definitions[]` ΓÇö Master array of media type definitions
- `dynamic_world` ΓÇö Game state including tick count
- `cosine_table[]`, `sine_table[]` ΓÇö Trig lookups for flow direction
- `GetMemberWithBounds()` ΓÇö Bounds-checked array accessor

# Source_Files/GameWorld/media.h
## File Purpose
Defines the media (hazard/liquid) system for the game world. Media represents volumes of liquid or hazardous substances (water, lava, goo, etc.) that affect gameplay through damage, currents, sounds, and visual effects. Provides data structures, enums, and function prototypes for media creation, updating, and querying.

## Core Responsibilities
- Define media types and enumerated constants (water, lava, goo, sewage, Jjaro goo)
- Store media properties: type, height, current direction/magnitude, texture, light binding
- Manage global media list (vector-based dynamic allocation)
- Provide accessors and utilities for media queries (danger, environment membership, sound/effect lookup)
- Support serialization/deserialization of media data (pack/unpack)
- Integrate with XML-based level parsing for liquids configuration

## External Dependencies
- **map.h** ΓÇô `SLOT_IS_USED` macro for media slot validity testing; constants like `WORLD_ONE`, `MAXIMUM_VERTICES_PER_POLYGON`
- **XML_ElementParser.h** ΓÇô `XML_ElementParser` base class for data-driven parsing
- **\<vector\>** ΓÇô STL dynamic array for `MediaList`
- **csmacros.h** (implied via includes) ΓÇô `TEST_FLAG16`, `SET_FLAG16` macros for bit manipulation

# Source_Files/GameWorld/media_definitions.h
## File Purpose
Defines media type properties for the Marathon/Aleph One game engine. Specifies visual representation, damage, detonation effects, and audio for five liquid/media types (water, lava, goo, sewage, jjaro).

## Core Responsibilities
- Define the `media_definition` structure for describing media properties
- Maintain a static array of media type definitions indexed by type
- Specify visual assets (collection, shape) and rendering mode for each media
- Configure damage properties (frequency, damage type) for damaging media
- Associate detonation effects for immersion/emergence events
- Map audio effects for interaction events and ambient sounds

## External Dependencies
- **Enums / constants (defined elsewhere):**
  - `_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro` (media type identifiers)
  - `_collection_walls1`, `_collection_walls2`, etc. (sprite/texture collections)
  - `_xfer_normal` (transfer/blend mode)
  - `_damage_lava`, `_damage_goo`, `_alien_damage` (damage type identifiers)
  - `_effect_*` constants (visual effect IDs)
  - `_snd_*`, `_ambient_snd_*` constants (sound IDs)
  - `NUMBER_OF_MEDIA_TYPES`, `NUMBER_OF_MEDIA_DETONATION_TYPES`, `NUMBER_OF_MEDIA_SOUNDS` (array sizes)
  - `FIXED_ONE`, `NONE` (special values)
- **Type dependencies:**
  - `damage_definition` struct (not defined in this file)
  - `int16` (standard type)

# Source_Files/GameWorld/monster_definitions.h
## File Purpose
Defines all monster types and their behavioral/physical attributes for the game engine. Contains data structures for monster definitions and attack parameters, along with initialization data for ~20+ distinct monster types, each with minor/major variants and special forms (invisible, kamikaze, tiny, etc.).

## Core Responsibilities
- Define monster class enumerations and class relationship macros (friends/enemies via bitfields)
- Define behavioral flags (omniscient, invisible, kamikaze, berserker, etc.)
- Define speed, intelligence, and door-handling difficulty constants
- Define `attack_definition` struct for melee and ranged attack parameters
- Define `monster_definition` struct (~128 bytes) encapsulating all monster attributes
- Declare global mutable and immutable monster definition arrays
- Provide serialization helpers for saving/loading monster definitions

## External Dependencies
- **Includes:** `effects.h` (for effect type enums like `_effect_fighter_blood_splash`), `projectiles.h` (for projectile type enums like `_projectile_staff`)
- **Defined elsewhere:** 
  - Macros: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `NUMBER_OF_ANGLES`, `BUILD_COLLECTION()`, `FLAG()`, `NONE`, `UNONE`
  - Types: `int16`, `uint32`, `uint8`, `_fixed`, `angle`, `world_distance`, `shape_descriptor`, `damage_definition`
  - Constants: `NUMBER_OF_MONSTER_TYPES`, effect/projectile type enums
  - Sounds: `_snd_fighter_activate`, `_snd_human_wail`, etc. (external sound IDs)

**Notes:**  
The file extensively uses C-style designated initializer syntax (comments labeling each field in aggregate initialization). Monster variants (minor/major/invisible/kamikaze/tiny) are separate type definitions; the `_class` bitfield enables friendly fire and faction logic. Attack shapes reference animation frame indices (frame numbers in collection sprites). The `carrying_item_type` field specifies what item a monster drops on death.

# Source_Files/GameWorld/monsters.cpp
## File Purpose
Implements the monster/NPC system for the game engine, including lifecycle management, AI behavior, pathfinding, combat, and serialization. Handles all non-player agents in the game world.

## Core Responsibilities
- **Monster lifecycle**: creation, activation, deactivation, and death
- **AI & targeting**: target selection, line-of-sight checks, mode/action transitions
- **Pathfinding**: cost-function-based path generation and navigation
- **Combat**: attack execution, damage calculation, projectile positioning
- **Physics & animation**: vertical/horizontal movement, collision detection, animation sequencing
- **Environmental interaction**: terrain features, platforms, media (liquid/hazards), doors
- **Serialization**: binary packing/unpacking of monster state and definitions
- **Configuration**: XML parsing for damage-kick parameters

## External Dependencies
- **Notable includes**: map.h, render.h, interface.h, effects.h, projectiles.h, player.h, platforms.h, scenery.h, SoundManager.h, items.h, media.h, flood_map.h, Packing.h, lua_script.h, monster_definitions.h
- **External globals**: monsters, monster_definitions, dynamic_world, static_world, objects, map_polygons, map_lines, map_endpoints, map_sides
- **External functions**: get_object_data(), get_polygon_data(), find_adjacent_polygon(), new_map_object(), remove_map_object(), animate_object(), play_object_sound(), new_effect(), delete_path(), flood_map(), find_closest_appropriate_target(), orphan_projectiles(), activate_nearby_monsters(), teleport_object_out(), L_Invalidate_Monster(), IsMediaDangerous()
- **Macros**: SLOT_IS_USED, MARK_SLOT_AS_USED, MONSTER_IS_ACTIVE, MONSTER_IS_DYING, etc. (from map.h and monsters.h)

# Source_Files/GameWorld/monsters.h
## File Purpose
Header for monster/NPC management in a Marathon-engine game. Defines data structures for tracking monster state, enums for types/actions/modes, and function prototypes for lifecycle, activation, targeting, collision detection, and damage systems.

## Core Responsibilities
- Define 46+ monster types (ticks, fighters, hummers, cyborgs, enforcers, etc.)
- Declare monster state struct (`monster_data`, 64 bytes) tracking vitality, position, action, mode, target
- Define bit-packed flags for active/idle/berserk/blind/deaf/recovering-from-hit status
- Declare monster lifecycle functions (spawn, despawn, activate, deactivate)
- Declare AI/targeting functions (find target, change target, lock/lose lock)
- Declare damage and collision detection (radius damage, melee, projectile impacts)
- Declare activation triggers based on proximity, sound, teleport, or editor bias
- Provide macros for querying and mutating monster state bits

## External Dependencies
- **dynamic_limits.h** ΓÇö `get_dynamic_limit()` for runtime limits on monster count, collision buffers
- **XML_ElementParser.h** ΓÇö `XML_ElementParser` base class for parsing monster definitions
- **cstypes.h** (implied) ΓÇö `int16`, `uint16`, `int32`, `world_distance`, `world_point3d`, etc.
- **\<vector\>** ΓÇö STL container for `MonsterList`
- **Constants:** `BUILD_DESCRIPTOR()` macro (collection/shape indices)

# Source_Files/GameWorld/pathfinding.cpp
## File Purpose
Implements path management and traversal for monster/NPC movement in the Marathon engine. Allocates a fixed pool of pre-computed paths, generates paths using flood-fill-based routing, and tracks movement along those paths for in-game entities.

## Core Responsibilities
- Allocate and manage a pool of reusable path objects
- Generate paths from source to destination polygons using flood-fill traversal
- Support both destination-driven (non-random) and random exploration paths
- Provide movement along paths with per-tick waypoint advancement
- Calculate safe waypoints on polygon boundaries accounting for minimum separation
- Debug inspection of path data (peek and validation)

## External Dependencies
- **Includes:**
  - `<string.h>`, `<stdlib.h>`, `<limits.h>` ΓÇö standard C library
  - `cseries.h` ΓÇö engine utilities (assert, obj_set, temporary buffer, csprintf)
  - `map.h` ΓÇö map geometry (polygon, line, endpoint accessors; find_shared_line, get_line_data, get_endpoint_data)
  - `flood_map.h` ΓÇö flood-fill pathfinding (flood_map, reverse_flood_map, flood_depth, choose_random_flood_node, cost_proc_ptr typedef)
  - `dynamic_limits.h` ΓÇö dynamic configuration (get_dynamic_limit, _dynamic_limit_paths enum)

- **Defined elsewhere:**
  - `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()` (flood_map.cpp)
  - `find_shared_line()`, `get_line_data()`, `get_endpoint_data()` (map accessors; map.cpp)
  - `get_dynamic_limit()` (dynamic_limits.cpp)
  - `global_random()` (random number generator; defined in game engine core)

# Source_Files/GameWorld/physics.cpp
## File Purpose
Implements player physics simulation for the Marathon/Aleph One game engine. Handles movement input processing, collision detection, gravity, jumping, falling, and camera positioning. Encapsulates physics model parameters (velocities, accelerations) and applies them each frame with proper collision response and external force handling.

## Core Responsibilities
- Initialize and update player physics state each tick
- Process action flags (movement, turning, jumping) into velocity/acceleration
- Perform collision detection against walls, objects, and ground
- Calculate and apply gravity, jumping, and climbing forces
- Manage player on-ground vs. airborne states
- Generate camera position with head bobbing and pitch/yaw
- Handle external forces (damage knockback, momentum)
- Pack/unpack physics constants for serialization

## External Dependencies
- **Includes**: `cseries.h` (common types, macros), `render.h`, `map.h`, `player.h`, `interface.h`, `monsters.h`, `media.h`, `ChaseCam.h`, `Packing.h`, `<string.h>`
- **External symbols used**:
  - `get_player_data()`, `get_monster_data()`, `get_object_data()` ΓÇô data accessors
  - `get_polygon_data()` ΓÇô map geometry
  - `get_media_data()` ΓÇô media (liquid) properties
  - `keep_line_segment_out_of_walls()`, `legal_player_move()`, `translate_map_object()`, `find_new_object_polygon()` ΓÇô collision/movement (defined elsewhere in map.c)
  - `changed_polygon()`, `monster_moved()`, `bump_monster()` ΓÇô game logic callbacks (defined elsewhere)
  - `cosine_table[]`, `sine_table[]` ΓÇô trigonometric lookups (defined elsewhere)
  - `isqrt()` ΓÇô integer square root (defined elsewhere)
  - `static_world`, `dynamic_world` ΓÇô global game state
  - `local_player` ΓÇô shortcut to local player data
  - `ChaseCam_IsActive()` ΓÇô camera mode check
  - `physics_models`, `original_physics_models` ΓÇô physics parameter tables (imported from physics_models.h)

# Source_Files/GameWorld/physics_models.h
## File Purpose
Defines physics model types and constants for character movement (walking/running) in the Aleph One game engine. Provides a struct to encapsulate all physics parameters and pre-initialized constant sets for different locomotion modes.

## Core Responsibilities
- Enumerate available physics models (walking, running)
- Define the `physics_constants` struct containing all character dynamics parameters (velocity limits, accelerations, angular motion, dimensions)
- Provide hardcoded initial physics constants for each model type
- Declare serialization/deserialization functions for physics constants (pack/unpack)
- Declare physics system initialization function

## External Dependencies
- `_fixed`: Fixed-point number type (defined elsewhere; used for physics precision)
- `FIXED_ONE`, `QUARTER_CIRCLE`: Macro constants (defined elsewhere; represent fixed-point 1.0 and ╧Ç/2)
- `uint8`: Standard integer type
- `size_t`: Standard size type

**Notes:** Comment references "1-2-3 Converter," suggesting legacy serialization support. File header notes 1991ΓÇô2001+ Bungie Studios and Aleph One developers; GPL 2.0 licensed.

# Source_Files/GameWorld/placement.cpp
## File Purpose
Manages the dynamic placement and respawning of monsters and items in the game world. Loads placement configuration from map files, places initial objects, periodically recreates objects based on difficulty/gameplay rules, and provides safe player spawn locations.

## Core Responsibilities
- Load placement frequency data (initial/min/max/random counts) for all monster and item types from map WAD
- Place initial objects at game start according to placement rules and predefined map locations
- Periodically recreate monsters and items to maintain minimum counts and random encounters
- Track current object counts and trigger replacement when objects are destroyed
- Validate polygon locations for safe object placement (avoid walls, other objects, players)
- Select random player spawn locations away from monsters and other players
- Manage asset loading (collections and sounds) for monsters that may be placed

## External Dependencies
**Notable includes / imports:**
- `cseries.h` ΓÇö core utilities, debug macros (`dprintf`, `assert`)
- `map.h` ΓÇö global world data (`dynamic_world`, `saved_objects`), geometry accessors, placement structs
- `monsters.h` ΓÇö `new_monster()`, `activate_monster()`, `find_closest_appropriate_target()`
- `items.h` ΓÇö `new_item()`
- `<string.h>` ΓÇö `string` functions (minimal use)

**Key external symbols (defined elsewhere):**
- `dynamic_world` ΓÇö global dynamic game state (object counts, tick counter)
- `saved_objects` ΓÇö map-editor-placed object locations
- `new_monster()`, `new_item()` ΓÇö object factories
- `activate_monster()`, `find_closest_appropriate_target()` ΓÇö monster behavior
- `global_random()` ΓÇö RNG
- `point_is_player_visible()`, `point_is_monster_visible()` ΓÇö visibility checks
- `get_polygon_data()`, `get_object_data()` ΓÇö map data accessors
- `GET_GAME_OPTIONS()` ΓÇö game settings macro
- Polygon/geometry utilities: `find_center_of_polygon()`, `find_new_object_polygon()`, `get_player_starting_location_and_facing()`

# Source_Files/GameWorld/platform_definitions.h
## File Purpose
Defines static configuration data for all platform types in the game world (doors and moving platforms). Initializes a lookup table mapping each platform type to its audio effects, movement properties, behavioral flags, and damage parameters.

## Core Responsibilities
- Define sound code constants for platform event types (starting, stopping, obstructed, uncontrollable)
- Declare the `platform_definition` structure that bundles audio, defaults, and damage data
- Populate static array `platform_definitions[]` with configuration for each platform type (8 types total)
- Provide indexed access to platform behavior at initialization and runtime

## External Dependencies
- **Defined elsewhere**: `static_platform_data` (struct), `damage_definition` (struct), sound constants (`_snd_*`), platform type constants (`_platform_is_*`), behavior flag constants, `NUMBER_OF_PLATFORM_TYPES` macro, `FLAG()` macro, `NONE` constant, `FIXED_ONE` constant

# Source_Files/GameWorld/platforms.cpp
## File Purpose
Implements platform (moving surface/door) simulation for the game world. Handles creation, activation, movement with collision detection, state management, and interaction with players and monsters. Includes serialization and XML configuration support.

## Core Responsibilities
- **Platform lifecycle**: creation (`new_platform`), initialization, state management
- **Physics simulation**: height changes, collision detection with obstruction handling, directional reversal
- **Interaction layer**: player/monster activation, entry/exit validation with pathfinding heuristics
- **Cascading control**: adjacent platform activation/deactivation based on static flags
- **Geometry updates**: endpoint/line heights, texture adjustments, media height tracking
- **Audio**: platform-specific sound effects (starting, stopping, obstructed, uncontrollable)
- **Serialization**: binary pack/unpack and XML configuration parsing

## External Dependencies
- **world.h**: `world_distance`, `world_point3d`, angle types
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `change_polygon_height()`, accessors
- **platforms.h**: Type definitions, flag macros
- **lightsource.h**: `set_light_status()`, light data
- **SoundManager.h**: Sound playback, `CauseAmbientSoundSourceUpdate()`
- **player.h**: `try_and_subtract_player_item()` for key requirements
- **media.h**: Media data and sound helpers
- **lua_script.h**: `L_Call_Platform_Activated()` hook
- **DamageParser.h**: Damage parsing for obstruction damage
- **XML_ElementParser.h**: XML configuration support

**Defined elsewhere**: `platform_definitions` array (from `platform_definitions.h`), `dynamic_world` global, `platforms` vector, `change_polygon_height()`, light/media/object accessors

# Source_Files/GameWorld/platforms.h
## File Purpose
Defines platform (moving doors and platforms) structures, enumerations, and behavior flags for the game world. Declares all platform-related functions and provides serialization utilities. Platforms are dynamic geometry pieces that can extend/contract, activate/deactivate, and interact with players and monsters.

## Core Responsibilities
- Define platform type enumeration (spht doors, pfhor doors, platforms, etc.)
- Define speed/delay preset enumerations for platform motion
- Define static configuration flags (27 bits) for platform properties
- Define dynamic runtime flags (10 bits) for platform state tracking
- Provide flag-testing and flag-setting macros for both static and dynamic flags
- Declare platform lifecycle functions (creation, activation, state changes)
- Declare platform-entity interaction queries (can monster enter/leave, can player use)
- Declare serialization/deserialization functions for save/load
- Declare XML parser accessor for loading platform definitions

## External Dependencies
- `XML_ElementParser.h` ΓÇö base class for XML parsing (platform definitions)
- Custom scalar types: `int16`, `uint16`, `uint32`, `world_distance`
- Macros: `TEST_FLAG16()`, `TEST_FLAG32()`, `SET_FLAG16()`, `SET_FLAG32()` ΓÇö defined elsewhere (flag manipulation utilities)
- Constants: `TICKS_PER_SECOND`, `WORLD_ONE` ΓÇö time and spatial units (defined elsewhere)
- Macro `MAXIMUM_VERTICES_PER_POLYGON` ΓÇö polygon geometry limit
- STL `vector<>` for dynamic platform list

# Source_Files/GameWorld/player.cpp
## File Purpose
Core player management system for the Aleph One game engine. Handles player lifecycle (creation, update, death/revival), state management (physics, inventory, powerups), damage/health, teleportation, serialization, and XML configuration of player mechanics. Approximately 2883 lines implementing the most frequently-updated game object type.

## Core Responsibilities
- Player creation, initialization, and teardown
- Per-tick player updates: physics, actions, powerups, oxygen mechanics
- Damage application, death sequences, respawning with penalty delays
- Weapon state management (intensity, decay)
- Oxygen depletion/replenishment based on environment (vacuum vs. normal air)
- Player shape/animation selection based on action state
- Teleportation logic (intra-level and inter-level)
- Powerup duration tracking and application (invisibility, invincibility, infravision, extravision)
- Item pickup and inventory management
- Player data serialization for save-games and network sync
- XML-driven configuration of player parameters and item definitions

## External Dependencies
- `map.h`, `world.h`: World geometry, polygons, coordinates
- `monsters.h`, `monster_definitions.h`: Monster class/damage types
- `weapons.h`, `items.h`: Weapon/item enums
- `SoundManager.h`, `fades.h`, `media.h`: Audio and visual effects
- `interface.h`, `game_window.h`: UI/view management
- `ActionQueues.h`, `Packing.h`: Input queues, serialization helpers
- `ChaseCam.h`, `lua_script.h`: Optional subsystems (chase camera, scripting)
- `XML_ElementParser.h`: Configuration parsing

# Source_Files/GameWorld/player.h
## File Purpose
Defines core player data structures, action flags, physics variables, and the API for managing player state in the Marathon/Aleph One engine. Covers player initialization, physics updates, weapons, damage, and persistent game state.

## Core Responsibilities
- Define `player_data` structure encapsulating all player state (location, energy, weapons, damage, inventory)
- Define `physics_variables` for per-player kinematics (position, velocity, heading, floor/ceiling heights)
- Manage player settings (initial energy, oxygen, visual range, team colors) via MML-configurable `player_settings_definition`
- Expose action flag bit definitions and macros for input encoding (movement, aiming, weapon triggers)
- Provide global player array and helper accessors (`local_player`, `current_player`)
- Declare player lifecycle functions (creation, deletion, per-tick updates)
- Declare physics and weapon update routines; pack/unpack for serialization

## External Dependencies

- **cseries.h** ΓÇô Base types, macros, memory utilities
- **world.h** ΓÇô Coordinate types (`world_point3d`, `angle`, `world_distance`, `_fixed`), trigonometry
- **map.h** ΓÇô World geometry accessors, damage definitions, game mode constants
- **XML_ElementParser.h** ΓÇô MML parser for runtime configuration
- **weapons.h** ΓÇô Weapon enums (`_weapon_pistol`, etc.) and weapon update routines
- **Implicit:** Monster/projectile managers (called during damage and update), platform/switch managers (called during action-key handling)

# Source_Files/GameWorld/projectile_definitions.h
## File Purpose
Defines the properties and behavior of all projectile types in the game. Serves as the central data repository for projectile metadata, including damage, visual effects, physics parameters, and behavioral flags. Initialized at game startup and referenced by the projectile system during runtime.

## Core Responsibilities
- Define projectile behavior flags (guidance, gravity, media penetration, melee properties, etc.)
- Declare the `projectile_definition` structure that encapsulates all projectile properties
- Initialize static and const arrays of projectile definitions for ~40+ projectile types
- Provide serialization interfaces for saving/loading projectile data
- Support both player and alien weapon projectiles with distinct behavior profiles

## External Dependencies
- **`damage_definition`** ΓÇö struct (defined elsewhere); embedded in each projectile definition
- **Effect/sound enums** ΓÇö `_effect_*`, `_snd_*` constants (defined elsewhere); index into game asset tables
- **`world_distance`** ΓÇö typedef for spatial measurements (likely fixed-point or scaled integer)
- **`_fixed`** ΓÇö typedef for sound pitch values (likely fixed-point number)
- **`BUILD_COLLECTION()`** ΓÇö macro (defined elsewhere); constructs collection indices
- **`NUMBER_OF_PROJECTILE_TYPES`** ΓÇö constant (defined elsewhere); array dimension
- **Damage type enums** ΓÇö `_damage_explosion`, `_damage_projectile`, `_alien_damage`, etc. (defined elsewhere)

---

**Notes:** The file is licensed under GPL and references the Marathon/Aleph One project (Bungie Studios). The projectile system supports ~40 types ranging from player weapons (rockets, grenades, SMG bullets) to alien weapons (compiler bolts, fusion bolts, hunter projectiles) and special effects (fist, ball, energy drain). Many projectiles support complex behaviors: media penetration, guided targeting, detonation cascades, and gravity variations.

# Source_Files/GameWorld/projectiles.cpp
## File Purpose
Manages the lifecycle and physics of all active projectiles in the game world. Handles projectile creation with initial conditions, per-tick movement and collision detection against world geometry and entities, and detonation with area-of-effect damage and visual effects.

## Core Responsibilities
- **Projectile creation** ΓÇö `new_projectile()` allocates and initializes projectiles with type, owner, velocity, and guided-target information
- **Per-tick simulation** ΓÇö `move_projectiles()` updates all active projectiles, applies gravity, detects collisions, and removes projectiles on detonation
- **Collision detection** ΓÇö `translate_projectile()` traces projectile path through map geometry and checks against monsters/scenery via spatial queries
- **Guided targeting** ΓÇö `update_guided_projectile()` computes steering corrections toward a target monster each tick
- **Detonation** ΓÇö `detonate_projectile()` applies area-of-effect damage and spawns visual effects
- **Lifecycle management** ΓÇö `remove_projectile()`, `orphan_projectiles()` (when owners die), `remove_all_projectiles()` (on level exit)
- **Media boundary handling** ΓÇö special case for projectiles with "penetrates media boundary" flag allowing water-surface interactions
- **Serialization** ΓÇö pack/unpack functions for save/restore and network synchronization

## External Dependencies
- **Geometry**: `map.h` (polygons, lines, endpoints, adjacent-polygon queries)
- **Entities**: `monsters.h`, `scenery.h`, `effects.h` (damage, visual effects)
- **Objects**: Map object creation/manipulation (`new_map_object3d`, `translate_map_object`, `remove_map_object`)
- **Projectile definitions**: `projectile_definitions.h` (imported; not defined here)
- **Sound**: `SoundManager.h` (flyby/rebound sounds)
- **Scripting**: `lua_script.h` (Lua detonation callbacks)
- **Utilities**: `cseries.h`, `Packing.h`, `dynamic_limits.h`

# Source_Files/GameWorld/projectiles.h
## File Purpose
Header for the projectile subsystem in the Aleph One game engine (Marathon-compatible). Defines data structures for active projectiles, enumerations of projectile types, and function prototypes for projectile creation, movement, collision detection, and cleanup.

## Core Responsibilities
- Enumerate all projectile types (rockets, grenades, bullets, energy weapons, etc.)
- Define the `projectile_data` structure for tracking active projectiles in the world
- Manage projectile lifecycle (creation, movement per frame, removal, detonation)
- Provide collision detection and obstruction reporting during projectile movement
- Handle projectile state flags (flyby events, damage causation)
- Serialize/deserialize projectile data and definitions for save/load
- Track owned vs. orphaned projectiles (whose owner died)

## External Dependencies
- **Includes:** `dynamic_limits.h` (for `get_dynamic_limit()`), `world.h` (for angle, world_point3d, world_distance types).
- **Defined elsewhere:** All function implementations in `PROJECTILES.C`; projectile definitions and sound resources (resource-fork or XML).
- **Types used:** `world_point3d`, `world_distance`, `angle`, `_fixed`, polygon indices, entity indices.

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

## External Dependencies

- **cseries.h** ΓÇö base types, macros (`NONE`, `TEST_FLAG`), assertions
- **map.h** ΓÇö `object_data`, `object_location`, `new_map_object()`, `get_object_data()`, `animate_object()`, `randomize_object_sequence()`, object owner/solidity flag macros
- **effects.h** ΓÇö `new_effect()`, effect type constants
- **ShapesParser.h** ΓÇö `Shape_GetParser()`, `Shape_SetPointer()`, shape descriptor utilities
- **scenery_definitions.h** ΓÇö `scenery_definitions[]` array, `NUMBER_OF_SCENERY_DEFINITIONS`
- Included but not visibly used: render.h, interface.h, flood_map.h, monsters.h, projectiles.h, player.h, platforms.h (likely for compilation dependencies)

**Utility function (defined elsewhere):**
- `GetMemberWithBounds()` ΓÇö bounds-checked array access with NULL return on out-of-range

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

## External Dependencies
- **`XML_ElementParser.h`**: Provides XML parsing base class for scenery configuration deserialization (Loren Petrich's modding framework)
- **Types from elsewhere:** `object_location`, `world_distance`, `short` (int16) indices into global scenery pool

# Source_Files/GameWorld/scenery_definitions.h
## File Purpose
Defines the properties, appearance, and behavior of static scenery objects (lamps, debris, containers, etc.) in the game world. Contains a static array of 61 scenery definitions organized by environment theme (lava, water, sewage, alien, Jjaro), each with collision radius, height, destruction effects, and state flags.

## Core Responsibilities
- Define scenery behavior flags (`_scenery_is_solid`, `_scenery_is_animated`, `_scenery_can_be_destroyed`)
- Declare the `scenery_definition` struct to store per-object metadata
- Populate static `scenery_definitions[]` array with 61 environment objects
- Provide lookup constant `NUMBER_OF_SCENERY_DEFINITIONS` for bounds
- Associate destruction effects (lamp breakage, explosions) with destroyable scenery

## External Dependencies
- **Macros (defined elsewhere)**: `BUILD_DESCRIPTOR()` ΓÇö constructs shape references
- **Type definitions (defined elsewhere)**: `shape_descriptor`, `world_distance`, `uint16`, `int16`
- **Constants (defined elsewhere)**: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `NONE`
- **Effect symbols (defined elsewhere)**: `_effect_lava_lamp_breaking`, `_effect_water_lamp_breaking`, `_effect_sewage_lamp_breaking`, `_effect_alien_lamp_breaking`, `_effect_grenade_explosion`
- **Collection symbols (defined elsewhere)**: `_collection_scenery1`, `_collection_scenery2`, `_collection_scenery3`, `_collection_scenery4`, `_collection_scenery5` ΓÇö texture/sprite atlases

# Source_Files/GameWorld/TickBasedCircularQueue.h
## File Purpose
Header-only template library implementing lock-free circular queues for tick-based game state buffering. Provides a single-reader, single-writer FIFO keyed by game tick, commonly used to queue action flags across game updates without explicit synchronization.

## Core Responsibilities
- Define a thread-safe circular queue abstraction for tick-keyed game elements (e.g., action flags)
- Provide abstract interface `WritableTickBasedCircularQueue` for queue writers
- Implement concrete queue `ConcreteTickBasedCircularQueue` with dual-tick (read/write) management
- Implement `DuplicatingTickBasedCircularQueue` to fan-out writes to multiple child queues
- Support safe concurrent reader/writer access without locks (via atomic int32 operations)
- Manage circular buffer with proper wrapping for negative and large tick values
- Allow post-enqueue element mutation via `MutableElementsTickBasedCircularQueue`

## External Dependencies
- **cseries.h** ΓÇô Provides `int32`, assert macros, and basic type definitions
- **std::set** ΓÇô Used in `DuplicatingTickBasedCircularQueue::mChildren` to track child queue pointers

# Source_Files/GameWorld/weapon_definitions.h
## File Purpose
Defines the complete weapon system for the game engine, including weapon classes, firing mechanics, shell casing physics, trigger properties, and static weapon configuration data. Serves as the authoritative definition for all in-game weapons and their combat parameters.

## Core Responsibilities
- Define weapon class enums (_melee_class, _normal_class, _dual_function_class, _twofisted_pistol_class, _multipurpose_class)
- Define weapon behavior flags (automatic fire, overload capability, media-traversal, trigger sharing, etc.)
- Define weapon animation states and collections (idle, firing, reloading, charging shapes)
- Declare shell casing physics (velocity, acceleration, ejection points)
- Define trigger mechanics per weapon (ammo type, fire rate, recoil, projectile type, burst patterns)
- Store weapon ordering and configurations as static/const arrays
- Provide serialization/deserialization functions for weapon definitions

## External Dependencies
- Type definitions: `int16`, `_fixed` (fixed-point), `world_distance`, `uint8` ΓÇö defined elsewhere
- Constants: `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `WORLD_ONE_FOURTH`, `NORMAL_WEAPON_DZ` ΓÇö defined elsewhere
- Enums (referenced as item types, sounds, projectiles, etc.): `_i_knife`, `_snd_magnum_firing`, `_projectile_fist`, etc. ΓÇö defined elsewhere
- Collection indices: `_weapon_in_hand_collection`, `_collection_player` ΓÇö defined elsewhere
- `NUMBER_OF_TRIGGERS` macro ΓÇö defined elsewhere (likely = 2 for primary/secondary)

# Source_Files/GameWorld/weapons.cpp
## File Purpose
Implements the weapon system for the Marathon game engine, managing player weapon states, firing mechanics, reloading, ammunition tracking, animation sequences, and shell casing visual effects. Supports dynamic weapon configuration via XML and handles serialization of weapon data for save/network games.

## Core Responsibilities
- Initialize and reset weapon state for players (new games, level transitions)
- Update weapon state machines each frame (firing, reloading, charging, idle animations)
- Handle trigger input and coordinate firing/reloading logic
- Manage ammunition counts and two-handed/dual-trigger weapon mechanics
- Generate and update shell casing visual effects
- Track weapon statistics (shots fired, shots hit)
- Render weapon display information for HUD and weapon sprite positioning
- Parse and apply XML weapon configuration (definitions, shell casings, weapon ordering)
- Pack and unpack weapon data to/from network/save-game streams
- Handle weapon luminosity contribution to player light intensity

## External Dependencies
- **Notable includes:** `cseries.h`, `map.h`, `projectiles.h`, `player.h`, `weapons.h`, `SoundManager.h`, `interface.h`, `items.h`, `monsters.h`, `game_window.h`, `Packing.h`, `shell.h`, `weapon_definitions.h`, `XML_ElementParser.h`
- **External symbols (defined elsewhere):**
  - `weapon_definitions` (array), `shell_casing_definitions` (array), `weapon_ordering_array` (array) ΓÇô from `weapon_definitions.h`
  - `dynamic_world`, `current_player_index` ΓÇô from `map.h` / `player.h`
  - `get_player_data()`, `get_item_kind()` ΓÇô from `player.h`, `items.h`
  - `new_projectile()`, `mark_projectile_collections()` ΓÇô from `projectiles.h`
  - `SoundManager::instance()->PlaySound()` ΓÇô from `SoundManager.h`
  - `GetMemberWithBounds()`, `objlist_clear()` ΓÇô from `cseries.h` / memory utilities
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadBoundedNumericalValue()` ΓÇô XML/string parsing utilities

# Source_Files/GameWorld/weapons.h
## File Purpose
Header file for the weapons subsystem in the Aleph One game engine. Defines weapon types, data structures for tracking weapon state and ammunition, and functions for initializing, updating, and managing player weapons throughout gameplay.

## Core Responsibilities
- Enumerate weapon types (_weapon_fist, _weapon_pistol, _weapon_plasma_pistol, etc.)
- Define data structures for weapon state, triggers, shell casings, and display information
- Declare initialization functions for the weapon manager and per-player weapons
- Implement weapon update logic called once per frame with player actions
- Handle item pickups for ammunition/reloading and weapon switching
- Manage weapon display positioning and animation for the 3D weapon model
- Provide serialization (packing/unpacking) for multiplayer/save support
- Support XML-based weapon definition configuration

## External Dependencies
- **Custom types:** `_fixed` (fixed-point integer), `uint16`, `uint32`, `uint8`, `size_t`
- **XML infrastructure:** `XML_ElementParser` (defined elsewhere, used in configuration)
- **Shell casing system:** `MAXIMUM_SHELL_CASINGS` constant exported to `lua_script.cpp` per inline comment
- **Weapon definition structure:** External binary size exposed as `SIZEOF_weapon_definition` for serialization without including the full definition
- **Resource management:** References weapon collections (collections are marked for load/unload but manager is not in this file)

# Source_Files/GameWorld/world.cpp
## File Purpose
Provides low-level mathematical primitives for the game world system, including trigonometric lookup tables, coordinate transformations (translation, rotation), distance calculations, angle arithmetic, and random number generation. Serves as the foundation for spatial operations and geometry computations throughout the engine.

## Core Responsibilities
- Build and maintain precomputed trigonometry lookup tables (sine, cosine, tangent) for efficient angle-based calculations
- Perform 2D and 3D point translations and rotations relative to origins
- Convert between coordinate systems via transformation matrices
- Compute distances between points (exact 3D, exact 2D, approximate 2D heuristic)
- Normalize angles into the valid range [0, NUMBER_OF_ANGLES)
- Calculate arctangent from x/y coordinates using binary search
- Generate pseudorandom numbers (global and local LFSR-based generators)
- Compute integer square roots via bit-by-bit algorithm
- Handle coordinate overflow for long-distance calculations beyond 16-bit range

## External Dependencies
- **stdlib.h**: `malloc()` (table allocation).
- **math.h**: `cos()`, `sin()`, `atan()` (used in `build_trig_tables()` to compute 2╧Ç).
- **limits.h**: `INT16_MAX`, `INT32_MIN` (range constants).
- **cseries.h**: Engine infrastructure and macros (type definitions).
- **world.h**: Game world type definitions (`world_point2d`, `world_point3d`, angle constants).

# Source_Files/GameWorld/world.h
## File Purpose
Core geometry and world-coordinate system definitions for the game engine. Defines fixed-point world coordinates, angles, trigonometric constants, and data structures for points, vectors, and 3D locations. Declares coordinate transformation, distance calculation, and random number generation functions.

## Core Responsibilities
- Define fixed-point world distance units and coordinate system constants
- Provide angle/trigonometric constants and normalization utilities
- Declare point and vector data structures (2D, 3D, fixed-point, and 64-bit long variants)
- Export trigonometric lookup table globals
- Declare coordinate transformation functions (rotation, translation, combined transform)
- Declare distance calculation functions (exact and approximate variants)
- Provide random number generation interface
- Support long-integer intermediate calculations for precision in distant geometry

## External Dependencies
- **Includes/typedefs used**: `int16`, `int32`, `uint16`, `uint32`, `_fixed`, `register` (defined elsewhere)
- **Global symbols defined elsewhere**: `cosine_table`, `sine_table` (WORLD.C)
- **Dependency**: `FIXED_FRACTIONAL_BITS` constant (likely fixed-point arithmetic header)
- **Function definitions**: All implementations in WORLD.C


