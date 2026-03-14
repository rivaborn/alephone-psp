# Source_Files/RenderMain/RenderVisTree.h

## File Purpose
Defines `RenderVisTreeClass`, a visibility tree structure extracted from `render.c` for determining polygon visibility from the viewer's perspective. Manages the tree traversal, polygon queueing, and screen-space clipping calculations needed for efficient rendering.

## Core Responsibilities
- Build and traverse a visibility tree rooted at the viewpoint polygon
- Track which polygons and sides are visible within the view cone
- Manage screen-boundary clipping data for vertices and line segments
- Calculate screen-space coordinates for visible endpoints
- Support ray-casting and polygon adjacency traversal
- Maintain polygon sort tree for depth ordering across visibility tree

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `line_clip_data` | struct | Screen clipping bounds and vectors for map lines |
| `endpoint_clip_data` | struct | Screen clipping bounds and vector for map endpoints |
| `clipping_window_data` | struct | Screen clipping window; linked-list node for multiple windows |
| `node_data` | struct | Visibility tree node: polygon reference, parent/child/sibling links, polygon-sort tree pointers, clipping data indices |
| `RenderVisTreeClass` | class | Main container managing tree nodes, clipping data, polygon queue, and view reference |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PolygonQueue` | `vector<short>` | member | Queue of polygon indices to process during tree traversal |
| `polygon_queue_size` | `size_t` | member | Working size of polygon queue |
| `line_clip_indexes` | `vector<size_t>` | member | Map-line index to line-clip index translation table |
| `Nodes` | `vector<node_data>` | member | All nodes in visibility tree; indexed references prevent stale-pointer bugs |
| `EndpointClips` | `vector<endpoint_clip_data>` | member | Clipping data for visible map endpoints |
| `LineClips` | `vector<line_clip_data>` | member | Clipping data for visible map lines |
| `ClippingWindows` | `vector<clipping_window_data>` | member | Screen clipping windows; linked-list structure |
| `endpoint_x_coordinates` | `vector<short>` | member | Screen x-coordinates for each visible endpoint |
| `view` | `view_data*` | member | Pointer to current view state (field-of-view, origin, pitch, yaw) |

## Key Functions / Methods

### build_render_tree
- **Signature:** `void build_render_tree()`
- **Purpose:** Constructs the visibility tree by traversing adjacent polygons and calculating visibility
- **Inputs:** (uses `view` member and map data)
- **Outputs/Return:** (builds `Nodes`, `PolygonQueue`, clipping vectors)
- **Side effects:** Populates tree node structure and clipping data
- **Calls:** `initialize_render_tree()`, `initialize_polygon_queue()`, `initialize_clip_data()`, `cast_render_ray()`, `PUSH_POLYGON_INDEX()`
- **Notes:** Entry point for visibility calculation; driven by polygon queue

### cast_render_ray
- **Signature:** `void cast_render_ray(long_vector2d *_vector, short endpoint_index, int ParentIndex, short bias)`
- **Purpose:** Cast ray from an endpoint to find adjacent visible polygons
- **Inputs:** Ray direction vector, endpoint index, parent tree node index (by index, not pointer), traversal bias
- **Outputs/Return:** (recursively builds tree structure)
- **Side effects:** Allocates new nodes in `Nodes` vector, queues polygons in `PolygonQueue`
- **Calls:** `next_polygon_along_line()`, `decide_where_vertex_leads()`, recursive calls
- **Notes:** Uses index-based parent reference to avoid stale-pointer bugs

### next_polygon_along_line
- **Signature:** `uint16 next_polygon_along_line(short *polygon_index, world_point2d *origin, long_vector2d *_vector, short *clipping_endpoint_index, short *clipping_line_index, short bias)`
- **Purpose:** Traverse along a line to find the next polygon that intersects or is adjacent
- **Inputs:** Current polygon, ray origin, direction vector, bias for tie-breaking
- **Outputs/Return:** Clipping indices; updates polygon_index to next polygon
- **Side effects:** None
- **Calls:** (geometric/vector operations, likely from world.h)
- **Notes:** States machine with four states (`_looking_for_first_nonzero_vertex`, etc.)

### Resize
- **Signature:** `void Resize(size_t NumEndpoints, size_t NumLines)`
- **Purpose:** Pre-allocate or grow internal vectors to accommodate endpoints and lines
- **Inputs:** Expected endpoint and line counts
- **Outputs/Return:** None
- **Side effects:** Resizes `endpoint_x_coordinates`, `EndpointClips`, `LineClips`, `ClippingWindows` (lazy resize)
- **Calls:** None visible
- **Notes:** Lazy resizing strategy; called before tree-building phase

### calculate_endpoint_clipping_information, calculate_line_clipping_information
- **Purpose:** Populate clipping structures for screen-boundary intersection
- **Side effects:** Append entries to `EndpointClips` or `LineClips`; update line/endpoint flag bits

### RenderVisTreeClass (constructor)
- **Purpose:** Initialize empty visibility tree
- **Side effects:** Initializes all member vectors and pointers

## Control Flow Notes
Visibility tree is built during the **render** phase, before polygon/side drawing. Root node is the viewpoint polygon. Tree is traversed via `build_render_tree()` ΓåÆ `cast_render_ray()` ΓåÆ `next_polygon_along_line()` ΓåÆ `decide_where_vertex_leads()` to populate visibility. Clipping calculations feed into screen-space transformation and rasterization. Polygon-sort tree (`PS_Greater`, `PS_Less`, `PS_Shared`) provides depth ordering for overlapping visible polygons.

## External Dependencies
- **Includes:** `<vector>` (STL), `world.h`, `render.h`
- **Defined elsewhere:**
  - `view_data` (render.h) ΓÇö view state with FOV, origin, pitch/yaw
  - `world_point2d`, `long_vector2d` (world.h) ΓÇö geometry primitives
  - Map polygon/endpoint/line data (via world.h)
  - `MAXIMUM_VERTICES_PER_POLYGON` constant (render.h)
