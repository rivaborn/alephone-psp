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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| flood modes enum | enum | Specifies traversal strategy: `_depth_first`, `_breadth_first`, `_flagged_breadth_first`, `_best_first` |
| cost_proc_ptr | typedef | Function pointer for custom edge cost calculation; takes source/dest polygon indices, line index, and caller data |

## Global / File-Static State
None.

## Key Functions / Methods

### flood_map
- Signature: `short flood_map(short first_polygon_index, long maximum_cost, cost_proc_ptr cost_proc, short flood_mode, void *caller_data)`
- Purpose: Execute a flood-fill pathfinding search from a starting polygon
- Inputs: Start polygon index, max cost threshold, cost function (NULL uses dest polygon area as default), flood mode, opaque caller data
- Outputs/Return: Path or query identifier (short)
- Side effects: Populates internal flood-map structures; allocates memory
- Calls: cost_proc (user-supplied callback)
- Notes: Default cost function (NULL) uses destination polygon area; breadth-first significantly faster than best-first for large domains

### new_path
- Signature: `short new_path(world_point2d *source_point, short source_polygon_index, world_point2d *destination_point, short destination_polygon_index, world_distance minimum_separation, cost_proc_ptr cost, void *data)`
- Purpose: Construct a path from source to destination with minimum separation constraint
- Inputs: Source/destination points and polygon indices, minimum separation distance, cost function, caller data
- Outputs/Return: Path handle (short)
- Side effects: Allocates path data

### move_along_path
- Signature: `bool move_along_path(short path_index, world_point2d *p)`
- Purpose: Advance one step along a previously created path
- Inputs: Path handle, output point buffer
- Outputs/Return: Boolean success

### reverse_flood_map, flood_depth, choose_random_flood_node
- Query recent flood-map results for depth information and random traversal nodes with optional bias weighting

## Control Flow Notes
Typical usage: `allocate_flood_map_memory()` ΓåÆ `flood_map()` ΓåÆ `new_path()` ΓåÆ `move_along_path()` in game loop ΓåÆ `delete_path()` and `reset_paths()` for cleanup. Flood-map executes during world pathfinding queries, supporting both AI navigation and grenade/projectile steering.

## External Dependencies
- `world_point2d`, `world_vector2d`, `world_distance` types (defined elsewhere; world geometry types)
- C standard types: `short`, `long`, `bool`, `void`
