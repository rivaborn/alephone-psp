# Source_Files/GameWorld/flood_map.cpp

## File Purpose
Implements a graph-traversal/flood-fill algorithm for pathfinding over a polygon-based game world map. Supports multiple search strategies (breadth-first, best-first) with iterative node expansion and path reconstruction via backtracking.

## Core Responsibilities
- Allocate and manage memory for the search node pool and visited-polygon tracker
- Execute one iteration of the selected search algorithm, returning the next polygon to expand
- Reconstruct paths by walking backward through the parent node chain
- Support cost-based node evaluation (best-first search) and optional caller-provided flags
- Select random nodes with optional directional bias for AI pathfinding

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Search tree node (16 bytes): tracks polygon index, cost, parent, depth, and expansion status |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `nodes` | `node_data*` | static | Array of search nodes (max 255) |
| `visited_polygons` | `short*` | static | Maps polygon index ΓåÆ node index for O(1) lookup |
| `node_count` | short | static | Current number of active nodes |
| `last_node_index_expanded` | short | static | Most recently expanded node (for backtracking and path reconstruction) |

## Key Functions / Methods

### allocate_flood_map_memory
- Signature: `void allocate_flood_map_memory(void)`
- Purpose: Allocate/reallocate memory for nodes and visited-polygon arrays
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates old arrays and allocates new ones (reentrant for map reloading)
- Calls: operator `new[]`, `delete[]`
- Notes: Called every time a map loads; asserts on allocation failure

### flood_map
- Signature: `short flood_map(short first_polygon_index, long maximum_cost, cost_proc_ptr cost_proc, short flood_mode, void *caller_data)`
- Purpose: Expand one node from the frontier, return its polygon index; on first call, initialize the search tree
- Inputs:
  - `first_polygon_index`: Starting polygon (NONE to continue existing search)
  - `maximum_cost`: Cost threshold for nodes
  - `cost_proc`: Optional callback to compute edge costs (NULL ΓåÆ use polygon area)
  - `flood_mode`: `_best_first`, `_breadth_first`, `_flagged_breadth_first`
  - `caller_data`: Context passed to cost_proc or flags storage
- Outputs/Return: Polygon index of next node to expand, or NONE
- Side effects: Updates `node_count`, `last_node_index_expanded`, nodes array, visited_polygons array
- Calls: `add_node()`, `get_polygon_data()`, `objlist_set()`
- Notes: Called iteratively; `_depth_first` asserts (unsupported); cost must be >0 to add

### reverse_flood_map
- Signature: `short reverse_flood_map(void)`
- Purpose: Return polygons along the path from destination back to source
- Inputs: None (uses global `last_node_index_expanded`)
- Outputs/Return: Parent polygon index, or NONE when path exhausted
- Side effects: Walks backward through `parent_node_index` chain
- Calls: None
- Notes: Call repeatedly after successful `flood_map()` to reconstruct path

### flood_depth
- Signature: `short flood_depth(void)`
- Purpose: Return polygon count from source to last expanded node
- Inputs: None
- Outputs/Return: Depth (distance in polygons)
- Side effects: None
- Calls: None

### choose_random_flood_node
- Signature: `void choose_random_flood_node(world_vector2d *bias)`
- Purpose: Select a random expanded node; if bias provided, prefer nodes in that direction
- Inputs: `bias` (optional direction vector)
- Outputs/Return: Updates global `last_node_index_expanded`
- Side effects: Global state modification
- Calls: `global_random()`, `find_center_of_polygon()`
- Notes: Requires node_count ΓëÑ 1; retries up to 10 times if bias provided

## Control Flow Notes
Typical usage pattern:
1. `allocate_flood_map_memory()` on level load
2. Call `flood_map(start_polygon, max_cost, ...)` to initialize
3. Loop `flood_map(NONE, ...)` until return value is NONE
4. Loop `reverse_flood_map()` to reconstruct path backward

The algorithm maintains a frontier of candidate nodes and expands them based on cost (best-first) or order (breadth-first). All adjacent non-solid polygons under the cost limit are added to candidates.

## External Dependencies
- **Includes**: `cseries.h` (core utilities, assert), `map.h` (polygon_data, world types), `flood_map.h` (public interface)
- **Defined elsewhere**: `get_polygon_data()`, `objlist_set()`, `find_center_of_polygon()`, `global_random()`, `dynamic_world`
- **Types**: `world_vector2d`, `world_point2d`, `polygon_data`, `cost_proc_ptr` (function pointer)
