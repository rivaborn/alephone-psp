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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `path_definition` | struct | Stores a single path: current step index, total step count, and array of up to 63 world points (waypoints) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `paths` | `path_definition*` | static | Dynamically-allocated array of all available paths; size set by `MAXIMUM_PATHS` from dynamic limits |
| `path_validation_area` | `byte*` | static | Debug buffer used when `VERIFY_PATH_SYNC` is defined; accumulates path point data across runs to detect inconsistencies |
| `path_validation_area_index` | `long` | static | Tracks current position in validation buffer |
| `path_run_count` | `short` | static | Counts how many times paths have been reset; used to switch validation modes |

## Key Functions / Methods

### allocate_pathfinding_memory
- **Signature:** `void allocate_pathfinding_memory(void)`
- **Purpose:** Allocate (or reallocate) the global path pool and validation buffer. Reentrant because it is called whenever `MAXIMUM_PATHS` changes via dynamic limits.
- **Inputs:** None (reads from `get_dynamic_limit(_dynamic_limit_paths)`)
- **Outputs/Return:** None
- **Side effects:** Frees and reallocates `paths` array; zeroes validation buffer if `VERIFY_PATH_SYNC` defined
- **Calls:** `get_dynamic_limit()` (from dynamic_limits), `new`, `delete`, `assert()`
- **Notes:** Uses `new`/`delete` for heap allocation; asserts on allocation failure

### reset_paths
- **Signature:** `void reset_paths(void)`
- **Purpose:** Mark all paths as free (unused) at the start of a new path-generation cycle. Called once per frame/tick before new paths are created.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets all `path_definition.step_count` to `NONE`; increments validation run counter
- **Calls:** None (loop-only)
- **Notes:** O(MAXIMUM_PATHS); enables pool reuse

### new_path
- **Signature:** `short new_path(world_point2d *source_point, short source_polygon_index, world_point2d *destination_point, short destination_polygon_index, world_distance minimum_separation, cost_proc_ptr cost, void *data)`
- **Purpose:** Generate a new path from source to destination polygon using flood-fill traversal. Supports both navigation-target paths (destination valid) and random exploration paths (destination invalid, bias provided via destination_point as a direction vector).
- **Inputs:**
  - `source_point`: 2D location (unused except in comments)
  - `source_polygon_index`: Polygon to start pathfinding from
  - `destination_point`: Target location (or bias vector if destination polygon is NONE)
  - `destination_polygon_index`: Target polygon (NONE for random path)
  - `minimum_separation`: Buffer distance from walls when computing waypoints
  - `cost`: Cost callback (or NULL for default area-based cost)
  - `data`: User context for cost callback
- **Outputs/Return:** Path index (0 to MAXIMUM_PATHSΓêÆ1) on success, or `NONE` if no free slot or path generation failed
- **Side effects:** Fills in a free `path_definition` slot with waypoints; updates `path_validation_area` if `VERIFY_PATH_SYNC` enabled
- **Calls:**
  - `flood_map()`, `reverse_flood_map()`, `flood_depth()`, `choose_random_flood_node()` (flood_map module)
  - `calculate_midpoint_of_shared_line()` (local)
  - `obj_set()` (macro for debug zeroing)
  - `memcmp()` (validation only)
  - Assertions for validation
- **Notes:**
  - Handles two modes: (a) reach destination, extract reverse-flood path; (b) random exploration within stack depth
  - Clips step count to `MAXIMUM_POINTS_PER_PATH` (63)
  - Handles edge case where depth is zero or path is too short
  - Waypoints are inserted in reverse order during path construction

### move_along_path
- **Signature:** `bool move_along_path(short path_index, world_point2d *p)`
- **Purpose:** Advance one step along a path and return the next waypoint. Called per-tick by entities following a path.
- **Inputs:**
  - `path_index`: Index of the path to advance
  - `p`: Output pointer to receive next waypoint coordinates
- **Outputs/Return:** `true` if end of path reached (and path is now freed), `false` if waypoint was successfully retrieved
- **Side effects:** Increments `current_step`; marks path as free (step_count = NONE) when exhausted
- **Calls:** None (direct array access)
- **Notes:** Asserts on invalid path index or step count; returns coordinates from `path->points[current_step++]`

### delete_path
- **Signature:** `void delete_path(short path_index)`
- **Purpose:** Manually free a path before it is naturally exhausted. Called when an entity abandons its path.
- **Inputs:** `path_index`: Index of path to free
- **Outputs/Return:** None
- **Side effects:** Sets `paths[path_index].step_count` to `NONE`
- **Calls:** Assertions only
- **Notes:** Asserts that path is in bounds and currently in use

### calculate_midpoint_of_shared_line (private)
- **Signature:** `static void calculate_midpoint_of_shared_line(short polygon1, short polygon2, world_distance minimum_separation, world_point2d *midpoint)`
- **Purpose:** Compute a safe waypoint on the line shared between two adjacent polygons, accounting for elevation endpoints and minimum wall separation.
- **Inputs:**
  - `polygon1`, `polygon2`: Adjacent polygon indices
  - `minimum_separation`: Buffer distance from endpoints
  - `midpoint`: Output pointer for computed waypoint
- **Outputs/Return:** Waypoint written to `*midpoint`
- **Side effects:** None (pure computation)
- **Calls:**
  - `find_shared_line()`, `get_line_data()`, `get_endpoint_data()` (map accessors)
  - `global_random()` (for offset randomization)
- **Notes:**
  - Finds the shared line between the two polygons
  - If endpoints are elevations, reduces usable range by `minimum_separation`
  - If range becomes zero, falls back to geometric midpoint of endpoints
  - Otherwise, computes random offset along the line, scaled by direction vectors

### path_peek (debug)
- **Signature:** `world_point2d *path_peek(short path_index, short *step_count)`
- **Purpose:** Inspect a path's waypoint array without advancing it. Debug/overhead-map function.
- **Inputs:** `path_index`, pointer to receive step count
- **Outputs/Return:** Pointer to waypoint array (or NULL if path is not in use); step count written to output
- **Side effects:** None
- **Calls:** None
- **Notes:** Returns null-check helper for callers

### GetNumberOfPaths
- **Signature:** `short GetNumberOfPaths()`
- **Purpose:** Query the total number of available path slots.
- **Inputs:** None
- **Outputs/Return:** `MAXIMUM_PATHS`
- **Side effects:** None
- **Calls:** `get_dynamic_limit()` macro expansion
- **Notes:** Convenience accessor; allows runtime inspection of pool size

## Control Flow Notes
**Initialization:** `allocate_pathfinding_memory()` is called during engine startup to allocate the path pool.

**Per-frame/tick:**
1. `reset_paths()` clears all paths to `NONE`
2. Entities (monsters) call `new_path(ΓÇª)` to request a path to their destination; pathfinding is performed once
3. On subsequent ticks, entities repeatedly call `move_along_path()` to retrieve the next waypoint and advance their position
4. When a path is exhausted or abandoned, `delete_path()` is called to free the slot
5. Freed slots become available for new entities

**Rendering/Debug:** `path_peek()` is used by the overhead map display to visualize monster paths.

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
