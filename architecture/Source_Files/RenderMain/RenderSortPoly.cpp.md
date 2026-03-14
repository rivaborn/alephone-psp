# Source_Files/RenderMain/RenderSortPoly.cpp

## File Purpose
Implements polygon depth-sorting for the render pipeline. Decomposes the render visibility tree into a depth-ordered list of sorted nodes, computing clipping windows for each polygon to constrain rendering boundaries.

## Core Responsibilities
- Sorts polygons into back-to-front rendering order via tree decomposition
- Builds clipping windows that define screen-space bounds for polygon rendering
- Accumulates clipping constraints from polygon parents and ancestors
- Calculates vertical clip data (top/bottom) covering specified horizontal runs
- Manages resizable STL vector containers for sorted nodes and clipping data

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `sorted_node_data` | struct | Represents a sorted polygon with its render objects and clipping windows |
| `clipping_window_data` | struct | Defines horizontal/vertical clipping bounds for a polygon (defined elsewhere) |
| `endpoint_clip_data` | struct | Left/right screen-edge clip markers (defined elsewhere) |
| `line_clip_data` | struct | Top/bottom horizontal clip markers (defined elsewhere) |
| `node_data` | struct | Render tree node with polygon index, children, parents, clipping info (from RenderVisTree) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MAXIMUM_SORTED_NODES` | `#define` | static | Initial reserve capacity (128) for STL vector of sorted nodes |
| `MAXIMUM_CLIPS_PER_NODE` | `#define` | static | Initial reserve capacity (64) for accumulated clip vectors |
| Window state enum | `enum` | static | Three states for clipping window FSM: `_looking_for_left_clip`, `_looking_for_right_clip`, `_building_clip_window` |

## Key Functions / Methods

### RenderSortPolyClass::RenderSortPolyClass()
- Signature: `RenderSortPolyClass()`
- Purpose: Initialize polygon sorter with null pointers and reserve vector capacity
- Inputs: None
- Outputs/Return: None (constructor)
- Side effects: Reserves 128 slots in `SortedNodes`, 64 slots in `AccumulatedEndpointClips` and `AccumulatedLineClips`
- Calls: STL `vector::reserve()`
- Notes: Idiot-proofs with NULL checks for `view` and `RVPtr`

### RenderSortPolyClass::Resize(size_t NumPolygons)
- Signature: `void Resize(size_t NumPolygons)`
- Purpose: Pre-allocate `polygon_index_to_sorted_node` mapping vector
- Inputs: `NumPolygons` ΓÇô count of polygons in the level
- Outputs/Return: None
- Side effects: Resizes `polygon_index_to_sorted_node` vector to match polygon count (lazy allocation)
- Calls: `vector::resize()`
- Notes: Maps polygon indices to their corresponding sorted nodes for fast lookup

### RenderSortPolyClass::sort_render_tree()
- Signature: `void sort_render_tree()`
- Purpose: Decompose the render visibility tree, extracting polygons in back-to-front order and building clipping info
- Inputs: `view` and `RVPtr` members (set externally)
- Outputs/Return: None (populates `SortedNodes`)
- Side effects: Clears and populates `SortedNodes`; modifies `polygon_index_to_sorted_node` mapping; destructively removes nodes from the tree
- Calls: `initialize_sorted_render_tree()`, `build_clipping_windows()`, polygon accessor functions
- Notes: Tree decomposition algorithmΓÇöpicks leaf polygons, checks for obstructions, removes them, repeats until root is removed. Uses binary search on `node_data::PS_Greater`/`PS_Less` pointers to find polygon instances. Handles vector reallocation by updating all dependent pointers.

### RenderSortPolyClass::build_clipping_windows(node_data *ChainBegin)
- Signature: `clipping_window_data *build_clipping_windows(node_data *ChainBegin)`
- Purpose: Construct clipping windows for a polygon by accumulating endpoint/line clips from the polygon and its ancestors
- Inputs: `ChainBegin` ΓÇô head of polygon-sharing node chain (nodes with same polygon index)
- Outputs/Return: Pointer to linked list of `clipping_window_data` (or NULL if none)
- Side effects: Clears and populates `AccumulatedEndpointClips` and `AccumulatedLineClips`; appends clipping windows to `RVPtr->ClippingWindows`; updates pointers in sorted nodes if vector reallocates
- Calls: `calculate_vertical_clip_data()`, `get_polygon_data()`, `TEST_RENDER_FLAG()`, `MAX()`/`MIN()`
- Notes: Finite-state machine tracks left/right clip pairs; discards duplicate clips; handles vector reallocation by patching next pointers. Adds screen edges (left, top, bottom, right) as initial bounds.

### RenderSortPolyClass::calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)
- Signature: `void calculate_vertical_clip_data(line_clip_data **accumulated_line_clips, size_t accumulated_line_clip_count, clipping_window_data *window, short x0, short x1)`
- Purpose: Compute top and bottom clip vectors for a window covering horizontal span `[x0, x1)`
- Inputs: `accumulated_line_clips` ΓÇô array of line clip pointers; `accumulated_line_clip_count` ΓÇô count; `window` ΓÇô target clipping window; `x0`, `x1` ΓÇô horizontal bounds
- Outputs/Return: None (populates `window->top`, `window->y0`, `window->bottom`, `window->y1`)
- Side effects: Writes to window struct; calls `vassert()` on invariant violations
- Calls: `vassert()`, `csprintf()`, comparison operators on line clip flags
- Notes: Scans left-to-right across `[x0, x1)` tracking highest top clip and lowest bottom clip; both must be continuous across the span. Uses `_clip_up` and `_clip_down` flags.

## Control Flow Notes
**Initialization phase:** Constructor reserves capacity; `Resize()` allocates the polygon-to-node map.  
**Per-frame rendering:** `sort_render_tree()` is called to decompose the visibility tree, populating `SortedNodes` in back-to-front order. For each polygon removed, `build_clipping_windows()` is invoked, which calls `calculate_vertical_clip_data()` to finalize window bounds.  
**Output:** Render pipeline consumes `SortedNodes` and associated clipping windows to rasterize polygons in depth order.

## External Dependencies
- **Includes:** `cseries.h` (macros, utilities), `map.h` (polygon_data, world structures), `RenderSortPoly.h` (class definition), `<string.h>`, `<limits.h>` (SHRT_MIN/MAX)
- **External symbols:** `get_polygon_data(short)`, `TEST_RENDER_FLAG()`, `TEST_FLAG()`, `csprintf()`, `vassert()`, `POINTER_CAST()`, `POINTER_DATA` (pointer arithmetic macros); `node_data`, `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` (defined in render.h); `RenderVisTreeClass` (defined in RenderVisTree.h); STL `std::vector` template
