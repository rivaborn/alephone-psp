# Source_Files/RenderMain/RenderRasterize.cpp

## File Purpose

Implements polygon clipping, coordinate transformation, and rasterization calls for a 3D game engine. Converts world-space polygon data into screen-space geometry, clips to view windows, and delegates textured rendering to a rasterizer backend. Handles floors, ceilings, walls, liquids, and sprite objects with proper depth ordering and lighting.

## Core Responsibilities

- Traverse sorted BSP/polygon tree and render visible geometry back-to-front
- Clip horizontal polygons (floors/ceilings) and vertical polygons (walls) to camera frustum
- Transform clipped vertices from world-space to perspective-correct screen-space
- Handle semi-transparent liquid surfaces with optional see-through rendering  
- Calculate and apply surface texturing, lighting, and transfer modes
- Clip sprite objects relative to liquid boundaries and view windows
- Support both 2D and 3D clipping algorithms (XY, Z, XZ planes)
- Translate animated textures and manage void-presence flags for transparency

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| flagged_world_point2d | struct | 2D world coordinate (int32 for long-distance) with clipping flags |
| flagged_world_point3d | struct | 3D world coordinate with clipping flags (Z is world_distance) |
| vertical_surface_data | struct | Temporary wall surface data: heights, endpoints, texture, lighting |
| RenderRasterizerClass | class | Main coordinate and rendering pipeline orchestrator |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MAXIMUM_VERTICES_PER_WORLD_POLYGON | #define | static | Max vertices after all clipping (base + 4) |
| view | pointer | member | Camera/view data; set by caller |
| RSPtr | pointer | member | Sorted polygon renderer; provides SortedNodes list |
| RasPtr | pointer | member | Backend rasterizer; performs texture_horizontal_polygon, texture_vertical_polygon, texture_rectangle |

## Key Functions / Methods

### render_tree
- Signature: `void RenderRasterizerClass::render_tree()`
- Purpose: Main entry point; iterate sorted polygons and render visible geometry in back-to-front order
- Inputs: None; uses member `RSPtr->SortedNodes` and view data
- Outputs/Return: None (side-effect rendering via `RasPtr`)
- Side effects: Calls `RasPtr` textured rendering methods; modifies screen buffer via backend
- Calls: `get_polygon_data()`, `get_media_data()`, `get_line_data()`, `get_side_data()`, `get_endpoint_data()`, `get_screen_mode()`, `TEST_FLAG()`, `Get_OGL_ConfigureData()`, `AnimTxtr_Translate()`, `render_node_floor_or_ceiling()`, `render_node_side()`, `render_node_object()`
- Notes:
  - Computes `SeeThruLiquids` flag based on OpenGL or software alpha-blending settings
  - Renders in order: ceiling ΓåÆ walls ΓåÆ objects (far side of liquid) ΓåÆ liquid surface ΓåÆ objects (near side)
  - Skips polygon if viewer is in wrong position relative to media boundary
  - Replaces ceiling/floor with media surface when appropriate and liquids are opaque

### render_node_floor_or_ceiling
- Signature: `void RenderRasterizerClass::render_node_floor_or_ceiling(clipping_window_data *window, polygon_data *polygon, horizontal_surface_data *surface, bool void_present)`
- Purpose: Render a horizontal surface (floor, ceiling, or liquid) after clipping and perspective transform
- Inputs: clipping window, source polygon (for vertex indices), surface properties, void_present flag
- Outputs/Return: None
- Side effects: Calls `RasPtr->texture_horizontal_polygon()`
- Calls: `AnimTxtr_Translate()`, `get_endpoint_data()`, `overflow_short_to_long_2d()`, `xy_clip_horizontal_polygon()` (x2), `z_clip_horizontal_polygon()` (x2), `get_shape_bitmap_and_shading_table()`, `get_light_intensity()`, `instantiate_polygon_transfer_mode()`
- Notes:
  - Builds vertex list from polygon endpoint indices; clips to window bounds (left/right, top/bottom, height)
  - Reverses vertex order for ceiling polygons to face camera correctly
  - Divides by zero guard: forces world_x to 1 if 0 to prevent division errors in perspective transform
  - Early exit if bitmap is NULL (nonexistent texture)

### render_node_side
- Signature: `void RenderRasterizerClass::render_node_side(clipping_window_data *window, vertical_surface_data *surface, bool void_present)`
- Purpose: Render a wall segment with height-based clipping and proper lighting
- Inputs: clipping window, wall surface (endpoints, heights, texture, lighting), void_present
- Outputs/Return: None
- Side effects: Calls `RasPtr->texture_vertical_polygon()`
- Calls: `AnimTxtr_Translate()`, `xy_clip_line()`, `xz_clip_vertical_polygon()` (x2), `get_shape_bitmap_and_shading_table()`, `get_light_intensity()`, `instantiate_polygon_transfer_mode()`
- Notes:
  - Constructs 4-vertex trapezoid from line posts and height bounds (h0, h1, hmax)
  - Clips endpoints in XY, then trapezoid in XZ (height dimension)
  - Calculates texture vectors (direction) and origin via fixed-point division
  - Divides by zero protection for divisor
  - Skips if height is zero (h Γëñ h0) or bitmap is missing

### render_node_object
- Signature: `void RenderRasterizerClass::render_node_object(render_object_data *object, bool other_side_of_media)`
- Purpose: Render sprite or 3D model with clipping relative to liquid surface
- Inputs: sprite/model object data, whether object is on opposite side of liquid from viewer
- Outputs/Return: None
- Side effects: Calls `RasPtr->texture_rectangle()`
- Calls: None (direct member access)
- Notes:
  - Sets clipping rectangle based on window bounds and liquid boundary (ymedia)
  - XOR logic: clips based on whether viewer and object are on same/opposite sides of media
  - 3D models: sets BelowLiquid flag instead of clipping clip_top/clip_bottom (handled elsewhere)
  - Iterates all clipping windows and sets rectangle bounds for each

### xy_clip_horizontal_polygon
- Signature: `short RenderRasterizerClass::xy_clip_horizontal_polygon(flagged_world_point2d *vertices, short vertex_count, long_vector2d *line, uint16 flag)`
- Purpose: Clip convex polygon to a line in 2D (XY plane) via cross-product containment testing
- Inputs: vertex array, count, clipping line direction, flag to mark clipped edges
- Outputs/Return: New vertex count (0 if fully clipped, >3 if expanded)
- Side effects: Modifies and rearranges vertex array; may call `memmove()`
- Calls: `SGN()`, `WRAP_HIGH()`, `WRAP_LOW()`, `xy_clip_flagged_world_points()`, `memmove()`
- Notes:
  - State machine: testing first vertex ΓåÆ searching for in/out transitions ΓåÆ computing intersection
  - Cross product sign: negative = inside (clipped), positive = outside (retained)
  - Uses bit-shifting for long-distance friendliness (CROSSPROD_TYPE)
  - Handles edge cases: all in (vertex_count=0), all out (no clipping), partial (insert 2 points)

### xy_clip_flagged_world_points
- Signature: `void RenderRasterizerClass::xy_clip_flagged_world_points(flagged_world_point2d *p0, flagged_world_point2d *p1, flagged_world_point2d *clipped, long_vector2d *line)`
- Purpose: Compute line-segment/clipping-plane intersection in 2D via parametric equation
- Inputs: segment endpoints p0, p1; clipping line vector
- Outputs/Return: Intersection point in clipped
- Side effects: Writes to clipped
- Calls: None
- Notes:
  - Swaps points to ensure consistent ordering (lower Y, then lower X)
  - Fixed-point division with bit-shifting to maintain precision (16 significant bits ratio)
  - Flags: intersection inherits AND of both input flags (common flags only)

### z_clip_horizontal_polygon / xz_clip_vertical_polygon
- Similar structure to `xy_clip_horizontal_polygon`; work on Z (height) and XZ (depth) dimensions respectively
- `z_clip_horizontal_polygon`: Clips 2D points at a fixed height; used for floors/ceilings
- `xz_clip_vertical_polygon`: Clips 3D points in XZ plane; used for walls

### xy_clip_line
- Signature: `short RenderRasterizerClass::xy_clip_line(flagged_world_point2d *posts, short vertex_count, long_vector2d *line, uint16 flag)`
- Purpose: Optimized clipping for 2-vertex line segment (faster than polygon clipping)
- Inputs: 2-element posts array, line vector, flag
- Outputs/Return: 0, 1, or 2 (vertices remaining)
- Side effects: May modify posts[0] or posts[1] via `xy_clip_flagged_world_points()`
- Calls: `xy_clip_flagged_world_points()`
- Notes: Three cases handled via cross product signs (both in, both out, one in one out)

### RenderRasterizerClass constructor
- Signature: `RenderRasterizerClass::RenderRasterizerClass()`
- Purpose: Initialize member pointers to NULL for idiot-proofing
- Side effects: Sets view, RSPtr, RasPtr to NULL
- Notes: Caller must set pointers before render_tree() call

## Control Flow Notes

Fits into render pipeline between `RenderSortPoly` (polygon sorting) and `Rasterizer` backend:

1. **Init**: `render_tree()` called with RSPtr and RasPtr set
2. **Per-polygon loop**: Iterate `RSPtr->SortedNodes` back-to-front
3. **Surface pass**: Render ceiling (if above viewer), then floor (if below)
   - Clip to window XY bounds, then Z bounds ΓåÆ transform to screen-space ΓåÆ rasterize
4. **Wall pass**: For each visible side, render full/high/low/transparent textures
   - Clip line endpoints in XY, trapezoid in XZ ΓåÆ transform to screen ΓåÆ rasterize
5. **Object pass**: Render sprites and 3D models
   - Clip relative to liquid boundary ΓåÆ rasterize rectangle
6. **Liquid pass** (if see-through): Render liquid surface after walls and far objects

## External Dependencies

- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `get_*_data()` accessors
- **lightsource.h**: `get_light_intensity()`
- **media.h**: `media_data`, `get_media_data()`
- **RenderRasterize.h**: Class definition, forward decls
- **AnimatedTextures.h**: `AnimTxtr_Translate()`
- **OGL_Setup.h**: `Get_OGL_ConfigureData()`, `TEST_FLAG()` macro
- **preferences.h**: `graphics_preferences` (alpha blending mode)
- **screen.h**: `get_screen_mode()`
- **csmacros.h** (via cseries.h): `TEST_RENDER_FLAG()`, `WRAP_HIGH()`, `WRAP_LOW()`, `SGN()`, `PIN()`, `FIXED_*` macros, `WORLD_ONE`
- **Rasterizer.h**: `RasterizerClass` with `texture_horizontal_polygon()`, `texture_vertical_polygon()`, `texture_rectangle()`
- **RenderSortPoly.h**: `RenderSortPolyClass` with `SortedNodes` vector
- **string.h**: `memmove()`

Defined elsewhere:
- `overflow_short_to_long_2d()`, `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (render utilities)
- `TEST_RENDER_FLAG()` (side visibility check)
- `CROSSPROD_TYPE`, `FIXED_FRACTIONAL_BITS`, `WORLD_FRACTIONAL_PART()` (fixed-point/coordinate types)
