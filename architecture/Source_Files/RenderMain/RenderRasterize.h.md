# Source_Files/RenderMain/RenderRasterize.h

## File Purpose
Defines the `RenderRasterizerClass`, which rasterizes (draws) visible world geometry to the screen. Performs coordinate transformation, viewport clipping, and delegates final rendering to a backend `RasterizerClass`. Part of the Aleph One game engine's rendering pipeline (after visibility tree construction and polygon depth-sorting).

## Core Responsibilities
- Render visible floors, ceilings, walls, and game objects to the framebuffer
- Clip world-space geometry to the viewport frustum (XY and Z bounds)
- Handle long-distance rendering by tracking coordinate overflow in flags
- Distinguish rendering for "void" vs. filled surfaces and media boundaries
- Coordinate between the visibility tree, sorted polygons, and object placement layers
- Transform and validate polygon vertex counts before rasterization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `flagged_world_point2d` | struct | 2D world point (int32 x, y) + clipping flags; used for floor/ceiling vertices |
| `flagged_world_point3d` | struct | 3D world point (int32 x, y, world_distance z) + clipping flags; used for wall vertices |
| `vertical_surface_data` | struct | Wall/side definition: texture, lighting, geometry (p0/p1 endpoints, heights h0/h1/hmax), transfer mode |

## Global / File-Static State
None.

## Key Functions / Methods

### render_tree
- **Signature:** `void render_tree()`
- **Purpose:** Main entry point; iterates through the sorted polygon tree and rasterizes all visible geometry (floors, ceilings, walls, objects).
- **Inputs:** None (uses member pointers: `view`, `RSPtr`, `RasPtr`).
- **Outputs/Return:** Geometry drawn via `RasPtr` (backend rasterizer).
- **Side effects:** Modifies framebuffer through rasterizer.
- **Calls:** `render_node_floor_or_ceiling()`, `render_node_side()`, `render_node_object()` (inferred from structure).
- **Notes:** Consumes data from `RenderSortPolyClass::SortedNodes`; processes sorted polygons in depth order.

### render_node_floor_or_ceiling (private)
- **Signature:** `void render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present)`
- **Purpose:** Rasterize a floor or ceiling polygon, with clipping and void handling.
- **Inputs:** Clipping window, polygon geometry, surface data (texture/lighting), void presence flag.
- **Outputs/Return:** Polygon drawn via rasterizer.
- **Side effects:** Modifies framebuffer; void_present flag affects transparency blending.
- **Notes:** Void-adjacent surfaces may skip semitransparency effects.

### render_node_side (private)
- **Signature:** `void render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present)`
- **Purpose:** Rasterize a wall/vertical surface with lighting and transfer mode.
- **Inputs:** Clipping window, wall data (p0/p1 endpoints, heights, texture, transfer mode), void flag.
- **Outputs/Return:** Wall drawn via rasterizer.
- **Side effects:** Applies ambient light delta and transfer mode.

### render_node_object (private)
- **Signature:** `void render_node_object(render_object_data *object, bool other_side_of_media)`
- **Purpose:** Rasterize a game object (sprite) with media boundary awareness.
- **Inputs:** Object data (rect, position, clipping), flag for rendering on opposite side of liquid surface.
- **Outputs/Return:** Object drawn via rasterizer.
- **Notes:** `other_side_of_media` flag affects how sprites render relative to water/lava surfaces.

### xy_clip_horizontal_polygon (private)
- **Signature:** `short xy_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Clip a horizontal polygon to the view cone in the XY plane.
- **Inputs:** Vertex array, count, clipping line, flag bits (_clip_left, _clip_right, etc.).
- **Outputs/Return:** Modified vertex count after clipping.
- **Calls:** `xy_clip_flagged_world_points()`.
- **Notes:** Uses long_vector2d for long-distance precision.

### z_clip_horizontal_polygon (private)
- **Signature:** `short z_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, world_distance height, uint16 flag)`
- **Purpose:** Clip a horizontal polygon to a Z height boundary (e.g., ceiling height).
- **Inputs:** Vertices, count, clipping line, height threshold, flag.
- **Outputs/Return:** Clipped vertex count.

### xz_clip_vertical_polygon (private)
- **Signature:** `short xz_clip_vertical_polygon(flagged_world_point3d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Clip a vertical (wall) polygon to the XZ plane.
- **Inputs:** 3D vertices, count, clipping line, flag.
- **Outputs/Return:** Clipped vertex count.

### xy_clip_line (private)
- **Signature:** `short xy_clip_line(flagged_world_point2d *posts, short vertex_count, long_vector2d *line, uint16 flag)`
- **Purpose:** Clip a line segment to the XY view cone (used for wall endpoints).
- **Inputs:** Endpoints, vertex count, clipping line, flag.
- **Outputs/Return:** Clipped vertex count.

### RenderRasterizerClass (constructor)
- **Signature:** `RenderRasterizerClass()`
- **Purpose:** Initialize the rasterizer; set up member pointers.
- **Notes:** Called once per frame or when switching render backends.

## Control Flow Notes
Part of a multi-stage rendering pipeline:
1. **RenderVisTree** constructs visibility tree from BSP world geometry.
2. **RenderSortPoly** sorts visible polygons by depth (nearest-to-farthest).
3. **RenderPlaceObjs** places game objects in rendering order.
4. **RenderRasterize** (this file) clips and rasterizes all geometry using the backend `RasterizerClass`.

Entry point is `render_tree()`; output is final framebuffer pixels.

## External Dependencies
- **world.h** ΓÇö coordinate types: `world_distance`, `long_vector2d`, `world_point3d`, `int32`
- **render.h** ΓÇö `view_data`, `polygon_data`, `clipping_window_data`
- **RenderSortPoly.h** ΓÇö `RenderSortPolyClass` (sorted polygons)
- **RenderPlaceObjs.h** ΓÇö `render_object_data` (game objects)
- **Rasterizer.h** ΓÇö `RasterizerClass` (abstract backend)
- **\<vector\>** ΓÇö STL for dynamic arrays
- **Defined elsewhere:** `render_object_data`, `polygon_data`, `horizontal_surface_data`, `clipping_window_data`
