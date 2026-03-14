# Source_Files/RenderMain/RenderPlaceObjs.h

## File Purpose
Defines the `RenderPlaceObjsClass` for spatial organization and depth-sorting of game objects (inhabitants) within the rendering pipeline. Objects are placed into a tree structure aligned with visible polygons and culled by visibility windows. Created by Loren Petrich; migrated from monolithic render.c to use STL vectors instead of custom growable lists.

## Core Responsibilities
- Build and populate a list of renderable objects with world positions and light intensities
- Construct render objects with calculated clipping windows for visibility culling
- Sort objects into a spatial tree keyed by polygon/node for depth ordering
- Calculate which polygons ("base nodes") are relevant to an object's position
- Rescale shape geometry based on distance and screen projection
- Integrate with the visibility tree (RenderVisTree) and polygon sorter (RenderSortPoly)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `render_object_data` | struct | Represents a single object ready to render: its containing node, calculated clipping bounds, linked-list chain, screen rectangle, and media Y-coordinate |
| `RenderPlaceObjsClass` | class | Container managing all objects in the scene; holds vectors of render objects and references to view data and related renderers |

## Global / File-Static State
None.

## Key Functions / Methods

### build_render_object_list
- **Signature:** `void build_render_object_list()`
- **Purpose:** Main public entry point; orchestrates creation and spatial organization of all game objects for the current frame
- **Inputs:** None (uses member pointers: `view`, `RVPtr`, `RSPtr`)
- **Outputs/Return:** None; populates `RenderObjects` vector
- **Side effects:** Clears and rebuilds entire object list; modifies `SortedNodes` in `RSPtr`
- **Calls:** `initialize_render_object_list()`, `build_render_object()` (per object), `sort_render_object_into_tree()`
- **Notes:** Called once per frame during the sort phase, after visibility tree and polygon sorting are complete

### build_render_object
- **Signature:** `render_object_data *build_render_object(long_point3d *origin, _fixed floor_intensity, _fixed ceiling_intensity, sorted_node_data **base_nodes, short *base_node_count, short object_index, float Opacity)`
- **Purpose:** Construct a single render object given world position, light values, and container nodes
- **Inputs:** World 3D position, floor/ceiling light intensities, array of base sorted nodes, node count, object index in world, opacity
- **Outputs/Return:** Pointer to newly created `render_object_data` (added to `RenderObjects` vector)
- **Side effects:** Allocates memory in vector; modifies `interior_objects` or `exterior_objects` chains in base nodes
- **Calls:** `build_base_node_list()`, `build_aggregate_render_object_clipping_window()`, `rescale_shape_information()`
- **Notes:** Assumes valid base_nodes array; sets up linked-list chains for fast traversal

### sort_render_object_into_tree
- **Signature:** `void sort_render_object_into_tree(render_object_data *new_render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Insert an object into the sorted polygon tree, linking it to the appropriate node chains (interior/exterior)
- **Inputs:** Object to insert, array of containing nodes, node count
- **Outputs/Return:** None
- **Side effects:** Modifies `interior_objects` and `exterior_objects` linked lists in sorted nodes
- **Calls:** (inferred) Direct list manipulation
- **Notes:** Determines whether object is interior or exterior to polygons; establishes render order

### build_base_node_list
- **Signature:** `short build_base_node_list(short origin_polygon_index, world_point3d *origin, world_distance left_distance, world_distance right_distance, sorted_node_data **base_nodes)`
- **Purpose:** Find all sorted nodes (polygons) near the object's position within a left/right distance range
- **Inputs:** Starting polygon index, world position, horizontal distance bounds
- **Outputs/Return:** Count of base nodes found; populates `base_nodes` array
- **Side effects:** Walks polygon graph; may traverse visibility tree
- **Calls:** (inferred) visibility/polygon tree traversal
- **Notes:** Result used to anchor object in spatial hierarchy; critical for culling and ordering

### build_aggregate_render_object_clipping_window
- **Signature:** `void build_aggregate_render_object_clipping_window(render_object_data *render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Calculate the union of clipping windows from all containing nodes to determine object visibility bounds on screen
- **Inputs:** Object to clip, its base nodes, node count
- **Outputs/Return:** None; populates `clipping_windows` in `render_object_data`
- **Side effects:** Allocates clipping window data
- **Calls:** (inferred) window merge/union logic
- **Notes:** Essential for avoiding off-screen object rendering

### rescale_shape_information
- **Signature:** `shape_information_data *rescale_shape_information(shape_information_data *unscaled, shape_information_data *scaled, uint16 flags)`
- **Purpose:** Transform shape metrics (hotpoints, bounds) based on distance/perspective scaling
- **Inputs:** Original shape info, output buffer, scaling flags
- **Outputs/Return:** Pointer to scaled shape info struct
- **Side effects:** Writes to scaled struct
- **Calls:** (inferred) scaling math
- **Notes:** Ensures sprites appear correct size at varying distances

## Control Flow Notes
This class executes in the per-frame render pipeline **after** `RenderVisTree` (visibility culling) and `RenderSortPoly` (polygon depth-sorting) complete. It populates object chains within each sorted node, enabling the final render loop to iterate objects in correct depth order. Objects are linked into `interior_objects` (closer to viewer) and `exterior_objects` (farther) chains.

## External Dependencies
- **STL:** `<vector>` 
- **Engine core:** `world.h` (3D coordinates, `long_point3d`, `world_point3d`, `world_distance`), `interface.h` (`shape_information_data`), `render.h` (`view_data`, `_fixed`), `RenderSortPoly.h` (`RenderSortPolyClass`, `sorted_node_data`, `clipping_window_data`)
- **External symbols** (not defined here): `view_data`, `sorted_node_data`, `clipping_window_data`, `RenderVisTreeClass`, `RenderSortPolyClass`, shape scaling/transformation logic
