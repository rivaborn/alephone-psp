# Source_Files/RenderOther/OverheadMapRenderer.cpp

## File Purpose
Implements the overhead map renderer for the Aleph One game engine. Transforms world geometry and objects into screen-space for the automap display, handling viewport-aligned rendering of polygons, lines, entities, annotations, and paths with color coding for different terrain and object types.

## Core Responsibilities
- Render visible polygons with terrain-type coloring (water, lava, platforms, hazard zones)
- Draw map lines with elevation and obstruction indicators
- Transform world coordinates to screen-space with viewport culling
- Render game objects (players, monsters, items, projectiles) with visibility filtering
- Display map annotations and the level name on the overhead view
- Generate and restore "false automaps" for checkpoint map rendering (flooded visibility)
- Manage path visualization for waypoint routes
- Respect game options for visibility of aliens, items, and projectiles

## Key Types / Data Structures
None defined in this file. All structures (`polygon_data`, `line_data`, `endpoint_data`, `media_data`, `platform_data`, `monster_data`, `player_data`, `map_annotation`, `object_data`, `map_object`) are defined elsewhere and accessed via external functions.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|--------------------------------|---------|
| `saved_automap_lines` | `byte*` | Class member (private) | Backup of automap line visibility for checkpoint map restoration |
| `saved_automap_polygons` | `byte*` | Class member (private) | Backup of automap polygon visibility for checkpoint map restoration |
| `dynamic_world` | Pointer (external) | Global | Runtime world state; accessed for polygon, line, endpoint, and object counts/data |
| `static_world` | Pointer (external) | Global | Level metadata; accessed for level name |
| `objects` | Array (external) | Global | Active game objects; iterated to render players, monsters, items, projectiles |
| `saved_objects` | Array (external) | Global | Initial object placements; used to locate checkpoint markers |
| `automap_lines` / `automap_polygons` | Byte arrays (external) | Global | Automap visibility flags; swapped during checkpoint map rendering |
| `local_player` | Pointer (external) | Global | Current player; used for team checks in omniscient mode |
| `ConfigPtr` | `OvhdMap_CfgDataStruct*` | Class member | Configuration for colors, shapes, display options |

## Key Functions / Methods

### Render
- **Signature:** `void OverheadMapClass::Render(overhead_map_data& Control)`
- **Purpose:** Main entry point; orchestrates the complete overhead map rendering pipeline including geometry, annotations, objects, and paths.
- **Inputs:** 
  - `Control`: overhead_map_data struct with origin (world position), scale, viewport dimensions
- **Outputs/Return:** None (side effects only)
- **Side effects (global state, I/O, alloc):** Modifies render flags on endpoints/polygons; calls virtual drawing methods; may allocate/deallocate temporary automap buffers; reads/writes game option flags
- **Calls (direct calls visible in this file):**
  - `begin_overall()`, `end_overall()` (virtual)
  - `GET_GAME_OPTIONS()`, `generate_false_automap()`, `replace_real_automap()`
  - `transform_endpoints_for_overhead_map()`
  - `begin_polygons()`, `end_polygons()`, `draw_polygon()` (virtual)
  - `begin_lines()`, `end_lines()`, `draw_line()` (virtual)
  - `get_polygon_data()`, `get_line_data()`, `get_media_data()`, `get_platform_data()`
  - `get_next_map_annotation()`, `draw_annotation()` (virtual), `draw_map_name()` (virtual)
  - `set_path_drawing()` (virtual), `path_peek()`, `GetNumberOfPaths()`, `draw_path()` (virtual), `finish_path()` (virtual)
  - `get_monster_data()`, `monster_index_to_player_index()`, `get_player_data()`, `draw_player()` (virtual), `draw_thing()` (virtual)
- **Notes:** 
  - Polygon coloring prioritized: platform (secret or normal) ΓåÆ ouch zones ΓåÆ hill ΓåÆ media (water/lava/goo/sewage/jjaro) ΓåÆ default
  - Line visibility depends on both clockwise and counterclockwise polygon visibility
  - Objects only rendered outside checkpoint map mode; checkpoint mode instead shows saved goal markers
  - Monsters and items flicker (drawn every other frame) based on `(dynamic_world->tick_count + i) & 8`
  - Uses macros `WORLD_TO_SCREEN()` to convert world coords; scale parameter is bitshift amount (0ΓÇô3 typically)

### transform_endpoints_for_overhead_map
- **Signature:** `void OverheadMapClass::transform_endpoints_for_overhead_map(overhead_map_data& Control)`
- **Purpose:** Transform all endpoint vertices from world-space to screen-space and set visibility flags for viewport culling.
- **Inputs:** 
  - `Control`: viewport origin, half-dimensions, scale
- **Outputs/Return:** None (modifies external state)
- **Side effects (global state, I/O, alloc):** Updates `endpoint->transformed` coordinates in all endpoint data; sets/clears `_endpoint_on_automap` render flags; propagates visibility to polygons via `_polygon_on_automap` flags
- **Calls (direct calls visible in this file):**
  - `get_endpoint_data()`, `SET_STATE_FLAG()`, `TEST_STATE_FLAG()`, `get_polygon_data()`
- **Notes:** 
  - Clips endpoints against screen bounds [0, screen_width] ├ù [0, screen_height]
  - Polygon is marked visible if **any** endpoint is visible (breadth-first visibility)
  - Render flags are persistent across calls

### generate_false_automap
- **Signature:** `void OverheadMapClass::generate_false_automap(short polygon_index)`
- **Purpose:** Save current automap state and generate a new one via flood-fill from a polygon, used for checkpoint map rendering.
- **Inputs:** 
  - `polygon_index`: Starting polygon for flood-fill visibility
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Allocates `saved_automap_lines` and `saved_automap_polygons`; backs up current automap buffers; zeroes the active automap; invokes `flood_map()` with custom cost function
- **Calls (direct calls visible in this file):**
  - `new` (memory allocation), `memcpy()`, `memset()`, `flood_map()`
  - `false_automap_cost_proc()` (callback)
- **Notes:** 
  - Buffer size calculated as `(count / 8 + (count % 8 ? 1 : 0)) * sizeof(byte)` (bit-packed)
  - Flood-fill uses breadth-first mode with `LONG_MAX` cost limit
  - Continues until no more polygons are reachable (loop until `flood_map()` returns `NONE`)

### replace_real_automap
- **Signature:** `void OverheadMapClass::replace_real_automap(void)`
- **Purpose:** Restore the saved automap buffers and free temporary allocations.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects (global state, I/O, alloc):** Deallocates `saved_automap_lines` and `saved_automap_polygons`; restores `automap_lines` and `automap_polygons` from backup
- **Calls (direct calls visible in this file):**
  - `memcpy()`, `delete[]`
- **Notes:** 
  - Null-checks before dereferencing; sets pointers to NULL after deletion
  - Must be paired with `generate_false_automap()` for checkpoint map mode

### false_automap_cost_proc
- **Signature:** `long OverheadMapClass::false_automap_cost_proc(short source_polygon_index, short line_index, short destination_polygon_index, void *caller_data)`
- **Purpose:** Cost function for `flood_map()`; prevents traversal into/out of secret platforms and adds traversed polygons/lines to the automap.
- **Inputs:** 
  - `source_polygon_index`: Polygon being left
  - `line_index`: Line being crossed (unused)
  - `destination_polygon_index`: Polygon being entered
  - `caller_data`: Unused
- **Outputs/Return:** Cost (1 = traversable, -1 = blocked)
- **Side effects (global state, I/O, alloc):** Modifies `automap_lines` and `automap_polygons` bitflags via macros `ADD_LINE_TO_AUTOMAP()` and `ADD_POLYGON_TO_AUTOMAP()`
- **Calls (direct calls visible in this file):**
  - `get_polygon_data()`, `get_platform_data()`, `PLATFORM_IS_SECRET()`, `PLATFORM_IS_DOOR()`, `ADD_LINE_TO_AUTOMAP()`, `ADD_POLYGON_TO_AUTOMAP()`
- **Notes:** 
  - Returns -1 (impassable) if source is a secret platform or destination is a secret door
  - Always adds source polygon and its lines to automap (even if cost is -1)
  - Static method to enable use as C-style callback for `flood_map()`

## Control Flow Notes
`Render()` is the frame/render-phase entry point. It orchestrates visibility determination (`transform_endpoints_for_overhead_map()`), then iterates world data in order: polygons ΓåÆ lines ΓåÆ annotations/paths ΓåÆ objects. Renders are delegated to virtual methods (`begin_*()`, `draw_*()`, `end_*()`), allowing subclasses to implement platform-specific drawing (2D rasterization, OpenGL, etc.). Checkpoint map mode diverts object rendering to show goal markers instead, using a temporary flooded automap generated and restored within the same `Render()` call.

## External Dependencies
- **Notable includes:** `cseries.h` (cross-platform compatibility), `OverheadMapRenderer.h` (class definition), `flood_map.h` (pathfinding), `media.h`, `platforms.h`, `player.h`, `render.h` (world rendering)
- **External symbols (defined elsewhere):**
  - World/game state: `dynamic_world`, `static_world`, `objects`, `saved_objects`, `local_player`
  - Data accessors: `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `get_media_data()`, `get_platform_data()`, `get_monster_data()`, `get_player_data()`
  - Automap state: `automap_lines`, `automap_polygons`
  - Pathfinding: `flood_map()`, `path_peek()`, `GetNumberOfPaths()`
  - Macros: `POLYGON_IS_IN_AUTOMAP()`, `TEST_STATE_FLAG()`, `SET_STATE_FLAG()`, `LINE_IS_IN_AUTOMAP()`, `LINE_IS_SOLID()`, `LINE_IS_VARIABLE_ELEVATION()`, `LINE_IS_LANDSCAPED()`, `WORLD_TO_SCREEN()`, `PLATFORM_IS_SECRET()`, `SLOT_IS_USED()`, etc.
  - Rendering hooks (virtual methods in subclass)
