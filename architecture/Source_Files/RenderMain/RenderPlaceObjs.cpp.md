# Source_Files/RenderMain/RenderPlaceObjs.cpp

## File Purpose
Manages placement of in-game objects (sprites, 3D models) into the sorted polygon rendering order for the Aleph One game engine. Handles 2D projection, depth sorting, clipping window calculation, and parasitic object linking to produce a renderable object list in depth order.

## Core Responsibilities
- Initialize and manage render object lists
- Transform world-space objects to screen space with 2D/3D projection
- Sort render objects into the rendering tree based on depth and polygon crossing
- Calculate aggregate clipping windows for objects spanning multiple polygons
- Project 3D model bounding boxes to sprite rectangles for proper depth sorting
- Handle parasitic objects (objects attached to host objects)
- Support object scaling (enlarged/tiny) and transfer modes (fade/fold/etc.)
- Integrate with OpenGL 3D model rendering and chase-cam opacity

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `render_object_data` | struct | Represents a renderable object: node reference, clipping windows, linked list, screen rectangle |
| `sorted_node_data` | struct | Polygon node in depth-sorted tree (defined in RenderSortPoly) |
| `shape_information_data` | struct | Shape rendering metadata: world extents (left/right/top/bottom), flags, minimum light intensity |
| `clipping_window_data` | struct | Screen clipping region with x/y bounds |
| `rectangle_definition` | struct | Screen rectangle with texture, shading, transfer mode, and model data |
| `RenderPlaceObjsClass` | class | Container managing render object list and rendering pipeline state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `FindProjectedBoundingBox` | function | static (file) | Projects 3D model bounding box to 2D sprite extents; calculates light position/direction for miner's light effect |
| `MAXIMUM_RENDER_OBJECTS` | macro | file | Recommended capacity for render object vector (72) |
| `MAXIMUM_OBJECT_BASE_NODES` | macro | file | Max sorted nodes an object can span (6) |
| `RenderObjects` | vector | class member | Vector of all render objects in current frame |
| `view` | pointer | class member | Current view data (camera, screen dimensions, projection) |
| `RVPtr` | pointer | class member | Render visibility tree (clipping windows) |
| `RSPtr` | pointer | class member | Render sorted polygons (depth-sorted node list) |

## Key Functions / Methods

### RenderPlaceObjsClass::RenderPlaceObjsClass()
- **Signature:** Constructor
- **Purpose:** Initialize render object manager with empty lists and null pointers
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reserves space in `RenderObjects` vector for performance
- **Calls:** `vector::reserve()`
- **Notes:** Idiot-proofing: sets view/RVPtr/RSPtr to NULL; must be set before use

### build_render_object_list()
- **Signature:** `void build_render_object_list()`
- **Purpose:** Main entry point; walks sorted polygon list and builds render objects for all in-world objects
- **Inputs:** None (uses class members: view, RSPtr, RVPtr)
- **Outputs/Return:** None (populates `RenderObjects`)
- **Side effects:** Clears and repopulates `RenderObjects`; modifies sorted node interior/exterior object pointers
- **Calls:** `initialize_render_object_list()`, `build_render_object()`, `build_aggregate_render_object_clipping_window()`, `sort_render_object_into_tree()`, `get_polygon_data()`, `get_light_intensity()`, `current_player->object_index`, `GetChaseCamData().Opacity`
- **Notes:** Iterates polygons in reverse order (back to front in depth); handles chase-cam opacity for self; skips invisible objects

### build_render_object()
- **Signature:** `render_object_data *build_render_object(long_point3d *origin, _fixed floor_intensity, _fixed ceiling_intensity, sorted_node_data **base_nodes, short *base_node_count, short object_index, float Opacity)`
- **Purpose:** Transform a single object to screen space and construct render rectangle; handles 3D model bounding-box projection
- **Inputs:** Optional origin (else uses object location); light intensities; polygon base nodes array; object index; opacity (chase-cam)
- **Outputs/Return:** Pointer to newly-constructed `render_object_data`, or NULL if invisible/too close/clipped/out of limit
- **Side effects:** Adds entry to `RenderObjects` vector; may reallocate vector and update all internal pointers; chains parasitic objects; modifies shape info if object is scaled
- **Calls:** `OBJECT_IS_INVISIBLE()`, `get_dynamic_limit()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `get_object_shape_and_transfer_mode()`, `rescale_shape_information()`, `extended_get_shape_information()`, `OGL_GetModelData()`, `FindProjectedBoundingBox()`, `build_base_node_list()`, `extended_get_shape_bitmap_and_shading_table()`, `instantiate_rectangle_transfer_mode()`, `get_light_intensity()`
- **Notes:** Handles vector reallocation by updating all stored pointers (render objects, sorted nodes); projects sprite Y/Z from transformed X; uses world_to_screen scaling factors; clamps to SHRT_MIN/SHRT_MAX; calculates liquid height relative to object; chains parasitic objects based on keypoint-obscured flag

### sort_render_object_into_tree()
- **Signature:** `void sort_render_object_into_tree(render_object_data *new_render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Insert render object(s) into depth-sorted linked list within appropriate sorted node
- **Inputs:** New render object (may be linked list of parasite); base nodes (polygons object spans); node count
- **Outputs/Return:** None (modifies object next_object pointers and sorted_node exterior_objects lists)
- **Side effects:** Reorders object linked lists; updates node references
- **Calls:** None directly visible (field access on render_object_data and sorted_node_data)
- **Notes:** Finds two boundary objects (deeper and shallower than new object); adjusts desired node based on boundaries; uses rectangle depth for comparison; handles parasites as single linked list

### build_base_node_list()
- **Signature:** `short build_base_node_list(short origin_polygon_index, world_point3d *origin, world_distance left_distance, world_distance right_distance, sorted_node_data **base_nodes)`
- **Purpose:** Determine which sorted nodes the object spans by ray-casting left and right from object origin
- **Inputs:** Origin polygon; object position; object left/right world extents
- **Outputs/Return:** Number of base nodes found (stored in base_nodes array)
- **Side effects:** None
- **Calls:** `get_polygon_data()`, `polygon_index_to_sorted_node[]`, `get_endpoint_data()`, `translate_point2d()`, `NORMALIZE_ANGLE()`, `TEST_RENDER_FLAG()`, cross-product calculations
- **Notes:** Ray-casts perpendicular to view yaw; uses vector cross-products to find polygon boundary crossings; skips polygons based on height (above/below viewer); limits to MAXIMUM_OBJECT_BASE_NODES

### build_aggregate_render_object_clipping_window()
- **Signature:** `void build_aggregate_render_object_clipping_window(render_object_data *render_object, sorted_node_data **base_nodes, short base_node_count)`
- **Purpose:** Merge clipping windows from all base nodes into one or more aggregate windows for the object
- **Inputs:** Render object; base nodes; node count
- **Outputs/Return:** None (sets render_object->clipping_windows)
- **Side effects:** Allocates new clipping_window_data entries; may reallocate `ClippingWindows` vector and update all node/object pointers
- **Calls:** `ClippingWindows.push_back()`, pointer arithmetic for reallocation fix-up
- **Notes:** Trivial case (one node): reuses node's clipping window; multi-node case: finds min/max Y bounds, sorts left/right x-values by depth, builds windows for each left/right pair

### rescale_shape_information()
- **Signature:** `shape_information_data *rescale_shape_information(shape_information_data *unscaled, shape_information_data *scaled, uint16 flags)`
- **Purpose:** Scale shape extents if object is enlarged (1.25├ù) or tiny (0.5├ù)
- **Inputs:** Unscaled shape info; flags (enlarged/tiny bits)
- **Outputs/Return:** Pointer to scaled or unscaled shape info
- **Side effects:** Writes to scaled structure if flags set
- **Calls:** None
- **Notes:** Idiot-proofing: returns NULL if unscaled is NULL; scales 6 world-distance fields (left/right/top/bottom/x0/y0)

### FindProjectedBoundingBox()
- **Signature:** Static function (not a method)
- **Purpose:** Project 3D model bounding box to screen space and calculate depth metrics for rendering
- **Inputs:** 3D bounding box [min/max][X/Y/Z]; transformed object position; scale; relative angle (object facing - view yaw); shape info; depth type; output references
- **Outputs/Return:** Fills Farthest (max X), ProjDistance (object center X), DistanceRef (reference X), LightDepth (miner's light depth), Direction (light direction unit vector)
- **Side effects:** Modifies ShapeInfo world_left/right/top/bottom
- **Calls:** `normalize_angle()`, `cosine_table[]`, `sine_table[]`, `isqrt()`, trigonometric/vector operations
- **Notes:** Rotates BB by object angle; expands to 8 vertices; finds projected Y/Z mins/maxs; uses DepthType to choose reference distance (farthest/nearest/center); calculates light direction from center to projected BB centroid; all in-file math, no external calls

## Control Flow Notes
**Rendering pipeline integration:**
- Called after sorted polygon visibility tree is constructed
- `build_render_object_list()` is the frame-entry point
- Iterates polygons in reverse depth order (back-to-front)
- Objects are transformed, projected, and linked into a per-node depth-sorted list
- Chase-cam opacity is applied to self-object
- 3D models replace sprite rectangles when present (OpenGL mode)
- Parasitic objects are chained after host in same polygon's list
- Clipping windows are computed per-object after all base nodes are known
- Output: `RenderObjects` vector with fully-initialized render_object_data entries, linked into sorted_node->interior/exterior_objects chains

## External Dependencies
- **Includes:** `map.h`, `lightsource.h`, `media.h`, `OGL_Setup.h`, `ChaseCam.h`, `player.h`, `RenderPlaceObjs.h`
- **Defined elsewhere:** `get_polygon_data()`, `get_light_intensity()`, `get_object_data()`, `get_endpoint_data()`, `get_object_shape_and_transfer_mode()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `rescale_shape_information()`, `instantiate_rectangle_transfer_mode()`, `OGL_GetModelData()` (conditional HAVE_OPENGL), `get_dynamic_limit()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `current_player`, `GetChaseCamData()`, trig tables (cosine_table, sine_table), `isqrt()`, `PIN()`, `normalize_angle()`
- **External types:** `view_data`, `sorted_node_data`, `polygon_data`, `object_data`, `endpoint_data`, `shape_information_data`, `clipping_window_data`, `rectangle_definition`, `OGL_ModelData`, `RenderVisTreeClass`, `RenderSortPolyClass`
