# Source_Files/RenderMain/RenderVisTree.cpp

## File Purpose
Implements a visibility-tree builder for the 3D rendering pipeline. The class traverses map geometry via ray-casting from a viewpoint, constructing a hierarchical tree of visible polygons and calculating clipping planes for efficient rendering of geometry and effects.

## Core Responsibilities
- Building a polygon visibility tree via recursive ray-casting from the camera viewpoint
- Queueing and processing polygons in breadth-first order
- Calculating screen-space clipping information (lines and endpoints that constrain rendering)
- Managing endpoint transformations and long-distance overflow handling
- Constructing a polygon-sorted tree (binary search structure) for efficient polygon lookups
- Detecting polygon transitions and elevation changes across transparent map boundaries
- Handling ray splits when rays pass through vertices exactly

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `node_data` | struct | Tree node representing a visible polygon; stores clipping constraints, parent/child/sibling pointers, and polygon-sort-tree links |
| `endpoint_clip_data` | struct | Clipping plane for a screen edge; stores screen coordinate and transformed vector |
| `line_clip_data` | struct | Clipping plane for a floor/ceiling edge; stores screen bounds and top/bottom vectors |
| `clipping_window_data` | struct | 2D clipping region defined by left/right and top/bottom vectors |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PolygonQueue` | `vector<short>` | instance member | Growable queue of polygon indices awaiting ray-casting |
| `polygon_queue_size` | `size_t` | instance member | Working size of polygon queue (smaller than capacity) |
| `Nodes` | `vector<node_data>` | instance member | Growable list of visibility-tree nodes; reallocates during recursion with pointer fixup |
| `EndpointClips` | `vector<endpoint_clip_data>` | instance member | Screen-edge clipping planes computed on-demand |
| `LineClips` | `vector<line_clip_data>` | instance member | Floor/ceiling clipping planes computed on-demand |
| `endpoint_x_coordinates` | `vector<short>` | instance member | Screen x-coordinates for all map endpoints (indexed by endpoint ID) |
| `line_clip_indexes` | `vector<size_t>` | instance member | Translation table from map line index to clip index |
| `view` | `view_data*` | instance member | Pointer to camera view parameters (set before calling `build_render_tree()`) |

## Key Functions / Methods

### build_render_tree
- **Signature:** `void RenderVisTreeClass::build_render_tree()`
- **Purpose:** Main entry point; builds the complete visibility tree by ray-casting from the viewpoint.
- **Inputs:** None (uses member `view` for camera parameters).
- **Outputs/Return:** None (updates member `Nodes`, `PolygonQueue`, clipping data).
- **Side effects:** Initializes polygon queue, render tree root, and clipping buffers. Sets render flags on polygons and endpoints. Modifies `Nodes` vector (may reallocate).
- **Calls:** `initialize_polygon_queue()`, `initialize_render_tree()`, `initialize_clip_data()`, `short_to_long_2d()`, `cast_render_ray()`, `get_polygon_data()`, `get_endpoint_data()`.
- **Notes:** Process loop iterates over queued polygons; for each, transforms untouched endpoints and casts rays. Stops when queue is empty. Handles long-distance transformations with overflow-point logic.

### cast_render_ray
- **Signature:** `void RenderVisTreeClass::cast_render_ray(long_vector2d *_vector, short endpoint_index, int ParentIndex, short bias)`
- **Purpose:** Recursively cast a ray through the map, creating/updating nodes for each polygon encountered.
- **Inputs:**
  - `_vector`: Ray direction in world space (long vector for precision).
  - `endpoint_index`: If NONE, no specific endpoint target; otherwise the solid/transparent endpoint we're aiming at.
  - `ParentIndex`: Index (not pointer) of parent node in `Nodes` vector; avoids stale-pointer bugs when vector reallocates.
  - `bias`: `_no_bias` (split at vertex), `_clockwise_bias`, or `_counterclockwise_bias` (choose next polygon).
- **Outputs/Return:** None (updates `Nodes`, clipping data).
- **Side effects:** Creates new `node_data` entries, updates parent/sibling links, may reallocate `Nodes` with pointer fixup. Sets/updates clipping info on nodes. Recursively calls itself on adjacent polygons.
- **Calls:** `next_polygon_along_line()`, `decide_where_vertex_leads()`, `calculate_line_clipping_information()`, `calculate_endpoint_clipping_information()`, `get_polygon_data()`.
- **Notes:** Implements a do-while loop traversing polygons along the ray. Detects ray splits and recursively casts clockwise/counterclockwise variants. Maintains parent-node pointer and index in sync; after vector reallocation, re-derives parent pointer from index. Polygon-sort-tree insertion is O(log n) on tree height.

### next_polygon_along_line
- **Signature:** `uint16 RenderVisTreeClass::next_polygon_along_line(short *polygon_index, world_point2d *origin, long_vector2d *_vector, short *clipping_endpoint_index, short *clipping_line_index, short bias)`
- **Purpose:** Given a polygon and a ray from origin, determine which edge the ray crosses and what polygon lies on the other side.
- **Inputs:**
  - `polygon_index`: Input/output; start polygon, updated to next polygon (or NONE).
  - `origin`: Ray origin in world space.
  - `_vector`: Ray direction.
  - `clipping_endpoint_index`: Input/output; if targeting a specific endpoint, used to detect if we pass through it.
  - `clipping_line_index`: Output; map line index crossed (NONE if no line).
  - `bias`: `_no_bias` (walk counterclockwise first), `_clockwise_bias`, `_counterclockwise_bias`.
- **Outputs/Return:** Clipping flags (`_clip_up`, `_clip_down`) indicating elevation changes; clipping indices updated.
- **Side effects:** Adds polygon to automap, sets render flags on polygons/lines/sides.
- **Calls:** `get_endpoint_data()`, `decide_where_vertex_leads()`, `get_line_data()`, `ADD_POLYGON_TO_AUTOMAP()`, `PUSH_POLYGON_INDEX()`, `ADD_LINE_TO_AUTOMAP()`, `SET_RENDER_FLAG()`, `LINE_IS_TRANSPARENT()`.
- **Notes:** State machine walks vertices (clockwise or counterclockwise) looking for edges where the cross-product changes sign (indicating a crossing). Handles degenerate cases (ray passes through a vertex) by splitting the ray. Resets loop-detection counters when state changes to avoid infinite loops.

### calculate_line_clipping_information
- **Signature:** `void RenderVisTreeClass::calculate_line_clipping_information(short line_index, uint16 clip_flags)`
- **Purpose:** Compute screen-space clipping vectors for a transparent line crossing (elevation change).
- **Inputs:**
  - `line_index`: Map line index.
  - `clip_flags`: `_clip_up` (ceiling change), `_clip_down` (floor change), or both.
- **Outputs/Return:** None (appends to `LineClips` vector).
- **Side effects:** Appends to `LineClips`; sets `_line_has_clip_data` flag and updates `line_clip_indexes[]`.
- **Calls:** `get_line_data()`, `get_endpoint_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `PIN()`, `SET_RENDER_FLAG()`.
- **Notes:** Transforms both endpoints; computes screen x-coordinates via perspective division. Calculates top/bottom vectors using ceiling/floor heights and world-to-screen scale. Pins screen coords to viewport; clears clip flags if off-screen.

### calculate_endpoint_clipping_information
- **Signature:** `short RenderVisTreeClass::calculate_endpoint_clipping_information(short endpoint_index, uint16 clip_flags)`
- **Purpose:** Compute screen-space clipping vector for an endpoint (vertex).
- **Inputs:**
  - `endpoint_index`: Map endpoint index (must have been transformed already).
  - `clip_flags`: `_clip_left` or `_clip_right` (exactly one).
- **Outputs/Return:** Index into `EndpointClips` vector, or NONE if endpoint was not transformed.
- **Side effects:** Appends to `EndpointClips`; sets `_endpoint_has_clip_data` flag.
- **Calls:** `TEST_RENDER_FLAG()`, `get_endpoint_data()`, `overflow_short_to_long_2d()`, `PIN()`.
- **Notes:** Returns early if endpoint not yet transformed; used to avoid clipping against untransformed vertices. Stores the clip vector negated for `_clip_right` to clip to the correct side.

### decide_where_vertex_leads
- **Signature:** `uint16 RenderVisTreeClass::decide_where_vertex_leads(short *polygon_index, short *line_index, short *side_index, short endpoint_index_in_polygon_list, world_point2d *origin, long_vector2d *_vector, uint16 clip_flags, short bias)`
- **Purpose:** Determine the next polygon when a ray passes through or near a vertex on a transparent line.
- **Inputs:**
  - `polygon_index`: Current polygon; updated to next.
  - `endpoint_index_in_polygon_list`: Index of the endpoint within the polygon's vertex list.
  - `bias`: `_no_bias` (split ray), `_clockwise_bias`, `_counterclockwise_bias`.
  - Other inputs: standard ray-casting parameters.
- **Outputs/Return:** Updated clipping flags (may add `_clip_left` or `_clip_right`).
- **Side effects:** Updates polygon/line/side indices; may set `_split_render_ray` flag.
- **Calls:** `get_polygon_data()`, `get_line_data()`, `get_endpoint_data()`, `LINE_IS_TRANSPARENT()`.
- **Notes:** If `_no_bias`, sets split flag and returns no next polygon. Otherwise, selects the adjacent polygon clockwise or counterclockwise and checks if we're leaving a solid endpoint (sets left/right clip flags).

### initialize_render_tree
- **Signature:** `void RenderVisTreeClass::initialize_render_tree()`
- **Purpose:** Initialize the visibility tree with a single root node pointing to the viewpoint's polygon.
- **Inputs:** None (uses `view->origin_polygon_index`).
- **Outputs/Return:** None (initializes `Nodes` vector with root node).
- **Side effects:** Clears `Nodes`; appends one dummy node and initializes it as root.
- **Calls:** `INITIALIZE_NODE()`.
- **Notes:** Root node has no parent, siblings, or children; all member pointers NULL.

### initialize_polygon_queue
- **Signature:** `void RenderVisTreeClass::initialize_polygon_queue()`
- **Purpose:** Reset the polygon queue for a new visibility tree build.
- **Inputs:** None.
- **Outputs/Return:** None (sets `polygon_queue_size` to 0).
- **Side effects:** Clears the active queue size (vector capacity unchanged).

### initialize_clip_data
- **Signature:** `void RenderVisTreeClass::initialize_clip_data()`
- **Purpose:** Initialize default clipping planes for screen edges (left, right, top, bottom).
- **Inputs:** None (uses `view` members).
- **Outputs/Return:** None (initializes `EndpointClips`, `LineClips`, `ClippingWindows`).
- **Side effects:** Calls `ResetEndpointClips()` and `ResetLineClips()`; clears `ClippingWindows`.
- **Notes:** Creates two default endpoint clips (screen edges) and one default line clip (top/bottom of screen).

### INITIALIZE_NODE (inline macro-turned-function)
- **Signature:** `inline void INITIALIZE_NODE(node_data *node, short node_polygon_index, uint16 node_flags, node_data *node_parent, node_data **node_reference)`
- **Purpose:** Initialize a freshly allocated node with default values.
- **Inputs:** Node pointer, polygon index, flags, parent pointer, reference pointer.
- **Outputs/Return:** None (modifies node in-place).
- **Side effects:** Sets all fields; initializes clipping lists to empty.
- **Notes:** All pointers (parent, siblings, children, PS_*) initialized to NULL.

### Resize
- **Signature:** `void RenderVisTreeClass::Resize(size_t NumEndpoints, size_t NumLines)`
- **Purpose:** Preallocate space in `endpoint_x_coordinates` and `line_clip_indexes` vectors.
- **Inputs:** Map endpoint and line counts.
- **Outputs/Return:** None (resizes vectors).
- **Side effects:** Lazy resizing; called before building the tree.

### RenderVisTreeClass (constructor)
- **Signature:** `RenderVisTreeClass::RenderVisTreeClass() : view(NULL)`
- **Purpose:** Construct and initialize an empty visibility-tree builder.
- **Inputs:** None.
- **Outputs/Return:** Instance with reserved vector capacities.
- **Side effects:** Reserves capacity (not size) for polygon queue, clips, and windows; sets `view` to NULL.

### ResetEndpointClips / ResetLineClips
- **Signature:** `void RenderVisTreeClass::ResetEndpointClips() / ResetLineClips()`
- **Purpose:** Clear and reinitialize the endpoint/line clipping lists with default screen-edge clips.
- **Inputs:** None (uses `view`).
- **Outputs/Return:** None (modifies clipping vectors).
- **Side effects:** Clears vector; pushes back initial dummy clip entries.

## Control Flow Notes
1. **Initialization Phase:** `build_render_tree()` calls initialization routines to set up empty structures.
2. **Root Ray-Casting:** Initial rays cast from the camera's left and right view edges toward a root polygon.
3. **Polygon Queue Loop:** Main loop pops polygons from the queue, transforms their unvisited endpoints, and casts rays at them.
4. **Recursive Ray-Casting:** `cast_render_ray()` walks along a ray, calling `next_polygon_along_line()` repeatedly to traverse adjacent polygons. When a polygon is first encountered, a node is created and added to the tree. Clipping information is accumulated.
5. **Ray Splitting:** When a ray hits a vertex (zero cross-product), it may split into two rays biased clockwise and counterclockwise; this is detected in `next_polygon_along_line()` via the state machine.
6. **Termination:** Tree building completes when the queue is empty (no more polygons to process).

## External Dependencies
- **cseries.h** ΓÇö Common series utilities (macros, types, assertions).
- **map.h** ΓÇö World geometry definitions (`polygon_data`, `endpoint_data`, `line_data`, `world_point2d`, world distance types, accessor functions).
- **render.h** ΓÇö Rendering declarations (included via RenderVisTree.h; defines `view_data`).
- **RenderVisTree.h** ΓÇö Class definition and helper macros (`WRAP_LOW`, `WRAP_HIGH`, `POINTER_CAST`, `CROSSPROD_TYPE`).
- **Functions defined elsewhere:** `get_polygon_data()`, `get_endpoint_data()`, `get_line_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `short_to_long_2d()`, `PIN()`, `SWAP()` (macro or inline).
- **Global flags/macros:** `TEST_RENDER_FLAG()`, `SET_RENDER_FLAG()`, `_polygon_is_visible`, `_endpoint_has_been_visited`, `_endpoint_has_been_transformed`, `_endpoint_has_clip_data`, `_line_has_clip_data`, `ENDPOINT_IS_TRANSPARENT()`, `LINE_IS_TRANSPARENT()`, `ADD_POLYGON_TO_AUTOMAP()`, `ADD_LINE_TO_AUTOMAP()`, `SET_RENDER_FLAG()`.
- **Constants:** `NONE` (likely -1 or sentinel), polygon/endpoint/line index enums.
