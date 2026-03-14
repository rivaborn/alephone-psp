# Source_Files/RenderMain/RenderSortPoly.h

## File Purpose
Defines a C++ class for sorting world polygons into appropriate depth order for rendering. Integrates with the visibility tree to organize sorted rendering nodes, manage clipping windows, and calculate vertex clipping data for the rasterization stage.

## Core Responsibilities
- Sort polygons into depth-order using visibility tree data
- Initialize and maintain the sorted render node tree structure
- Build and manage clipping windows for polygon culling
- Calculate vertical clipping bounds for rendering edges
- Accumulate and resize internal clip data structures
- Provide mapping from map polygon indices to sorted nodes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sorted_node_data` | struct | Represents a sorted polygon node: polygon index, interior/exterior render objects, and associated clipping windows |
| `RenderSortPolyClass` | class | Main sorting engine; orchestrates polygon depth-ordering and clipping window construction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `polygon_index_to_sorted_node` | `vector<sorted_node_data *>` | member | Maps from map polygon indices to their sorted node pointers (valid when `_polygon_is_visible`) |
| `SortedNodes` | `vector<sorted_node_data>` | member | Growable list of sorted polygon nodes; populated in `initialize_sorted_render_tree()` and `sort_render_tree()` |
| `AccumulatedEndpointClips` | `vector<endpoint_clip_data *>` | member | Accumulation list for endpoint clip data used during window building |
| `AccumulatedLineClips` | `vector<line_clip_data *>` | member | Accumulation list for line clip data used during window building |
| `view` | `view_data *` | member | Pointer to the current view being rendered |
| `RVPtr` | `RenderVisTreeClass *` | member | Pointer to the visibility tree object |

## Key Functions / Methods

### RenderSortPolyClass (constructor)
- **Purpose:** Initialize the polygon sorter with default state
- **Inputs:** None
- **Outputs/Return:** Initialized object
- **Side effects:** Allocates or resets internal vectors
- **Notes:** Lazy initialization pattern; actual data structures sized on first use

### Resize
- **Signature:** `void Resize(size_t NumPolygons)`
- **Purpose:** Resize all internal vectors to accommodate the given number of polygons; lazy sizing
- **Inputs:** `NumPolygons` ΓÇö maximum polygon count from the map
- **Outputs/Return:** (void)
- **Side effects:** Reallocates `polygon_index_to_sorted_node` and related vectors
- **Calls:** (vector allocation)
- **Notes:** Called when map is loaded or changed; design permits repeated resize calls

### sort_render_tree
- **Signature:** `void sort_render_tree()`
- **Purpose:** Main entry point; sorts visible polygons into depth order using the visibility tree
- **Inputs:** Uses member `RVPtr` (visibility tree) and `view` (view context)
- **Outputs/Return:** Populates `SortedNodes` vector
- **Side effects:** Modifies `SortedNodes`; updates clipping window pointers
- **Calls:** (not visible in header; defined elsewhere)
- **Notes:** Relies on visibility tree having been built first; called each frame

### initialize_sorted_render_tree
- **Signature:** `void initialize_sorted_render_tree()` (private)
- **Purpose:** Reset and prepare the sorted node tree for a new sort pass
- **Inputs:** (none; uses member state)
- **Outputs/Return:** (void)
- **Side effects:** Clears or resets `SortedNodes`
- **Calls:** (not visible)

### build_clipping_windows
- **Signature:** `clipping_window_data *build_clipping_windows(node_data *ChainBegin)` (private)
- **Purpose:** Construct clipping window chain from a node's clipping data
- **Inputs:** `ChainBegin` ΓÇö root node in a clipping chain from the visibility tree
- **Outputs/Return:** Pointer to the built clipping window (or chain)
- **Side effects:** Allocates clipping windows; updates `AccumulatedEndpointClips` and `AccumulatedLineClips`
- **Calls:** `calculate_vertical_clip_data()`

### calculate_vertical_clip_data
- **Signature:** `void calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)` (private)
- **Purpose:** Compute vertical screen-space clipping bounds for a horizontal span
- **Inputs:** Accumulated line clips, target window, x-coordinate range (`x0`, `x1`)
- **Outputs/Return:** (void)
- **Side effects:** Modifies clipping window structure
- **Notes:** Used during clipping window construction to populate top/bottom bounds

## Control Flow Notes
Called during the per-frame render pipeline **after** visibility determination (via `RenderVisTreeClass`) but **before** rasterization. The sequence is:
1. Visibility tree builds which polygons and surfaces are visible from the viewpoint
2. `sort_render_tree()` takes that visibility information and sorts polygons by depth
3. Clipping windows organize screen-space bounds for each polygon
4. Later rendering code draws polygons using the sorted depth order and clipping guidance

## External Dependencies
- **Includes:** `<vector>` (STL), `world.h` (world coordinate types), `render.h` (view/render flags), `RenderVisTree.h` (visibility tree definition)
- **Defined elsewhere:** `view_data` (render.h), `node_data` (RenderVisTree.h), `render_object_data` (not shown), `clipping_window_data` (RenderVisTree.h), `line_clip_data` / `endpoint_clip_data` (RenderVisTree.h)
