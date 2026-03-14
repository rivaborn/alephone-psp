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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `endpoint_data` | struct | Map vertex; 16 bytes; stores position, flags (elevation, solidity, transparency), supporting polygon |
| `line_data` | struct | Map edge connecting two endpoints; 32 bytes; references adjacent polygons and their side indices |
| `side_data` | struct | Surface (wall/floor/ceiling) facing a polygon; ~64 bytes; holds textures, transfer modes, control panel data |
| `polygon_data` | struct | Enclosed area (room); 128 bytes; stores vertices, heights, media, lights, neighbor/exclusion data; type enum specifies behavior (teleporter, trigger, etc.) |
| `object_data` | struct | Runtime game entity; 32 bytes; position, shape, animation state, flags (rendered, solid, owner type), transfer mode |
| `map_object` | struct | Initial placement record; 16 bytes; type (monster/item/object/player/goal), location, polygon index, facing |
| `damage_definition` | struct | Damage parameters; 12 bytes; type, base/random amounts, flags (difficulty scaling) |
| `game_data` | struct | Game configuration; game type, options, difficulty, kill limits, cheat flags |
| `dynamic_data` | struct | Runtime game state; 604 bytes; tick count, entity counts, random seed, player state, object/monster/projectile/effect tallies |
| `static_data` | struct | Level properties; 88 bytes; environment flags, physics model, music, mission type, entry points |
| `ambient_sound_image_data` | struct | Non-directional ambient sound; 16 bytes; sound index, volume, flags |
| `random_sound_image_data` | struct | Directional/pulsing sound effect; 32 bytes; sound, volume envelope, period envelope, pitch, phase |
| `map_annotation` | struct | Automap label; 72 bytes; text, type (color/font), location, polygon visibility |
| `saved_static_light_data` | struct | Light source definition; 100 bytes; light type, animation specs for active/inactive states, tag |
| `entry_point` | struct | Level entry point; mission/environment flags, level name |
| `player_start_data` | struct | Player spawn; team, identifier (encodes auto-recenter/weapon-switch flags), color, name |
| `object_frequency_definition` | struct | Random spawning rule; 12 bytes; initial/min/max counts, random chance, reappearance flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `static_world` | `struct static_data*` | global | Points to current level's static properties (environment, mission, entry flags) |
| `dynamic_world` | `struct dynamic_data*` | global | Points to current level's runtime state (tick count, entity counts, game info) |
| `ObjectList` | `vector<object_data>` | global | All runtime objects in current level; accessed via `objects` macro |
| `EndpointList`, `LineList`, `SideList`, `PolygonList` | `vector<*_data>` | global | Geometry data; accessed via `map_endpoints`, `map_lines`, `map_sides`, `map_polygons` macros |
| `AmbientSoundImageList`, `RandomSoundImageList` | `vector<*_image_data>` | global | Sound placement data |
| `MapAnnotationList`, `SavedObjectList` | `vector<map_annotation>`, `vector<map_object>` | global | Automap labels, initial object placements |
| `MapIndexList`, `AutomapLineList`, `AutomapPolygonList` | `vector<int16>`, `vector<uint8>` | global | Index buffer, automap visibility bitsets |
| `game_is_networked` | bool | global | True if running a network game |
| `LandscapesLoaded` | bool | global | Whether Marathon 2 landscapes were loaded (M1 compatibility) |
| `LoadedWallTexture` | short | global | Index of first loaded wall texture; used for infravision |

## Key Functions / Methods

### initialize_marathon
- **Signature:** `void initialize_marathon(void);`
- **Purpose:** Engine startup; initializes trig tables, random seed, memory allocation for map structures
- **Calls:** (defined elsewhere; called before any level)
- **Notes:** Called once at program start

### entering_map
- **Signature:** `bool entering_map(bool restoring_saved);`
- **Purpose:** Prepare a level for play; load geometry, objects, sound/light data; initialize NPC spawn
- **Inputs:** `restoring_saved` ΓÇô if true, skip Pfhortran initialization (post-load skip)
- **Outputs:** true if successful
- **Notes:** Called when transitioning to a new level; counterpart is `leaving_map()`

### update_world
- **Signature:** `std::pair<bool, int16> update_world(void);`
- **Purpose:** Main frame update; advance tick count, move monsters/projectiles, animate objects, update lights/sounds
- **Outputs:** pair of (whether state changed, real-mode elapsed ticks) ΓÇô prediction may request rerender even if ticks=0
- **Side effects:** Modifies `dynamic_world`, object positions, animation states
- **Notes:** Core game loop; called once per frame; supports prediction lookahead

### new_map_object2d / new_map_object3d / new_map_object
- **Signature:** `short new_map_object2d(world_point2d *location, short polygon_index, shape_descriptor shape, angle facing);` etc.
- **Purpose:** Spawn a new object at a map location
- **Inputs:** position (2D or 3D), polygon, shape, facing angle
- **Outputs:** object index or NONE if allocation failed
- **Side effects:** Adds to ObjectList; links into polygon's object list; calls `object_was_just_added()`
- **Notes:** 2D version sets z=0; 3D version uses full coordinate; generic version takes `object_location` struct

### remove_map_object
- **Signature:** `void remove_map_object(short index);`
- **Purpose:** Despawn an object; clean up links and counts
- **Inputs:** object index
- **Side effects:** Unlinks from polygon; marks slot free; calls `object_was_just_destroyed()`

### translate_map_object
- **Signature:** `bool translate_map_object(short object_index, world_point3d *new_location, short new_polygon_index);`
- **Purpose:** Move an object to a new position and/or polygon
- **Inputs:** object index, new 3D location, new polygon
- **Outputs:** true if move succeeded (coordinate valid, polygon found)
- **Side effects:** Updates object's position and polygon; relinks into new polygon's object list

### changed_polygon
- **Signature:** `void changed_polygon(short original_polygon_index, short new_polygon_index, short player_index);`
- **Purpose:** Called when player enters a new polygon; activates triggers (lights, platforms, teleporters)
- **Inputs:** original and new polygon indices, player index
- **Side effects:** May change polygon heights, activate devices, teleport player

### update_world (coordinate queries)
- **Signature:** `short world_point_to_polygon_index(world_point2d *location);`
- **Purpose:** Find which polygon contains a 2D point
- **Inputs:** 2D coordinate
- **Outputs:** polygon index or NONE
- **Notes:** Core for collision and pathfinding

### point_in_polygon
- **Signature:** `bool point_in_polygon(short polygon_index, world_point2d *p);`
- **Purpose:** Test if 2D point is inside a specific polygon (exclusion zone aware)
- **Inputs:** polygon index, point
- **Outputs:** true if point is inside polygon bounds and not in exclusion zones

### find_adjacent_polygon / find_adjacent_side / find_shared_line
- **Signature:** `short find_adjacent_polygon(short polygon_index, short line_index);` etc.
- **Purpose:** Traverse geometry: find polygon across a line, side definition, or shared line between two polygons
- **Inputs:** indices of reference entities
- **Outputs:** index of adjacent/shared entity or NONE
- **Notes:** Support polygon flood-fill and visibility queries

### distance calculations
- **Signature:** `world_distance distance2d(world_point2d *p0, world_point2d *p1);`, `int32 point_to_line_distance_squared(world_point2d *p, world_point2d *a, world_point2d *b);` etc.
- **Purpose:** Euclidean distance (2D/3D) and point-to-line distance for collision/AI
- **Inputs:** two points, or point and line segment
- **Outputs:** distance (or squared distance to avoid sqrt)
- **Calls:** `isqrt()` for integer square root

### play_object_sound / play_polygon_sound / play_world_sound
- **Signature:** `void play_object_sound(short object_index, short sound_code);` etc.
- **Purpose:** Queue sound for playback from object, polygon, or world position
- **Inputs:** sound code; object or polygon index (or origin point for world sound)
- **Side effects:** Updates sound system; may adjust pitch based on distance/doppler

### change_polygon_height
- **Signature:** `bool change_polygon_height(short polygon_index, world_distance new_floor_height, world_distance new_ceiling_height, struct damage_definition *damage);`
- **Purpose:** Dynamically change floor/ceiling height (platform/moving geometry)
- **Inputs:** polygon index, new heights, optional damage to apply
- **Outputs:** true if successful
- **Side effects:** Updates polygon geometry; may crush entities; triggers adjacent polygon updates

### change_light_state / change_device_state
- **Signature:** `void change_light_state(size_t lightsource_index, short state);`, `void change_device_state(short device_index, bool active);`
- **Purpose:** Toggle lights and devices (control panels, platforms) in response to player action or triggers
- **Inputs:** light/device index, state
- **Side effects:** Updates light animation, device status; may change platform heights or control panel status

### Pack/Unpack functions (serialization)
- **Signature:** `uint8 *pack_polygon_data(uint8 *Stream, polygon_data* Objects, size_t Count);` (and similarly for other types)
- **Purpose:** Serialize/deserialize map data to/from byte stream for save/load
- **Inputs:** byte stream (or output buffer), array of objects, count
- **Outputs:** pointer to next byte in stream (for chaining)
- **Notes:** Handle endianness and padding; every major data struct has pack/unpack pair

## Control Flow Notes

**Initialization phase** (startup to level entry):
- `initialize_marathon()` sets up engine-wide state (trig tables, memory)
- `generate_map(level_index)` or `load_level_from_map(level_index)` loads geometry and objects
- `entering_map(false)` places initial objects, activates entry-point lights/platforms, spawns player

**Frame loop** (repeating):
- `update_world()` ticks game clock, moves entities, animates objects, updates lights/sounds
- Collision detection and event handling (platform activation, damage application) occur within update
- Render system queries geometry and objects via accessor macros

**Event-driven** (throughout):
- `changed_polygon()` fires when player/object crosses into a new polygon; triggers teleporters, areas, devices
- `try_and_toggle_control_panel()` handles player interaction with switches/panels
- `handle_random_sound_image()` updates positional/pulsing ambient sounds
- `cause_polygon_damage()` applies area damage (lava, suffocation, etc.)

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
