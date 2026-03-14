# Source_Files/RenderMain/render.cpp

## File Purpose
Core rendering pipeline orchestrator for the Aleph One game engine. Manages view initialization, coordinates the multi-stage rendering process (visibility, sorting, object placement, rasterization), handles camera effects, and renders the player's HUD/weapons layer. Serves as the main entry point between the game world and graphics output.

## Core Responsibilities
- Memory allocation and initialization for rendering system (`allocate_render_memory`)
- View/camera parameter setup and per-frame updates (`initialize_view_data`, `update_view_data`)
- Main rendering loop orchestration (`render_view`), dispatching to specialized subsystems
- Render effect application (earthquakes, teleport folds) with phase tracking
- Transfer mode instantiation for polygons and sprites (texture animations, fades, static, etc.)
- First-person weapon/HUD rendering (`render_viewer_sprite_layer`)
- Sprite positioning and viewport calculations

## Key Types / Data Structures
None defined in this file. Relies on external types: `view_data`, `bitmap_definition`, `rectangle_definition`, `polygon_definition`, `weapon_display_information`, `shape_information_data`.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|---------|---------|
| RenderFlagList | `vector<uint16>` | global | Bit flags for visibility/clipping state of geometry elements |
| RenderVisTree | `RenderVisTreeClass` | static | Builds visibility tree (which polygons visible from viewpoint) |
| RenderSortPoly | `RenderSortPolyClass` | static | Sorts polygons by depth; accumulates clipping windows |
| RenderPlaceObjs | `RenderPlaceObjsClass` | static | Determines visible objects and places them in sorted order |
| RenderRasterize | `RenderRasterizerClass` | static | Clips and rasterizes each object to screen |
| Rasterizer_SW | `Rasterizer_SW_Class` | static | Software rasterizer implementation |
| Rasterizer_OGL | `Rasterizer_OGL_Class` | static (ifdef HAVE_OPENGL) | OpenGL rasterizer implementation |

## Key Functions / Methods

### allocate_render_memory
- **Signature:** `void allocate_render_memory(void)`
- **Purpose:** Allocate and initialize all rendering subsystem memory at startup.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Resizes `RenderFlagList`; resizes and links rendering classes.
- **Calls:** `vector::resize()`, setter methods on render classes.
- **Notes:** Sets up pointer cross-references between subsystems (e.g., `RenderSortPoly.RVPtr = &RenderVisTree`).

### initialize_view_data
- **Signature:** `void initialize_view_data(struct view_data *view)`
- **Purpose:** Calculate screen-to-world projection parameters from FOV and screen dimensions.
- **Inputs:** `view` ΓÇö view structure with `field_of_view`, `screen_width`, `screen_height`, `standard_screen_width` set.
- **Outputs/Return:** Fills `view` fields: `half_cone`, `half_vertical_cone`, `world_to_screen_x/y`, edge vectors, `landscape_yaw`.
- **Side effects:** Modifies `view` in-place.
- **Calls:** `asin()`, `tan()`, `atan()`, `View_FOV_FixHorizontalNotVertical()`.
- **Notes:** Handles aspect ratio adjustments; uses trigonometric overflowing to ensure clip boundaries lie off-screen.

### render_view
- **Signature:** `void render_view(struct view_data *view, struct bitmap_definition *destination)`
- **Purpose:** Main rendering orchestrator; executes the complete render pipeline each frame.
- **Inputs:** `view` ΓÇö current camera state; `destination` ΓÇö target framebuffer.
- **Outputs/Return:** None.
- **Side effects:** Clears render flags; modifies automap state; writes to `destination`.
- **Calls:** `update_view_data()`, `RenderVisTree.build_render_tree()`, `RenderSortPoly.sort_render_tree()`, `RenderPlaceObjs.build_render_object_list()`, `RenderRasterize.render_tree()`, `render_viewer_sprite_layer()`, `render_computer_interface()`, `render_overhead_map()`.
- **Notes:** Selects rasterizer (OpenGL or software) at runtime; skips 3D rendering if overhead map or terminal mode is active.

### update_view_data
- **Signature:** `static void update_view_data(struct view_data *view)`
- **Purpose:** Update camera vectors and clipping planes each frame based on yaw, pitch, and active effects.
- **Inputs:** `view` with updated `yaw`, `pitch`, `origin`, and effect state.
- **Outputs/Return:** None.
- **Side effects:** Modifies `view` edge vectors, `dtanpitch`, and media boundary state.
- **Calls:** `View_AdjustFOV()`, `update_render_effect()`, `get_polygon_data()`, `get_endpoint_data()`, `UNDER_MEDIA()`.
- **Notes:** Insets camera origin if sitting on polygon vertex (prevents ray-casting artifacts); determines if under liquid.

### update_render_effect
- **Signature:** `static void update_render_effect(struct view_data *view)`
- **Purpose:** Apply per-frame updates to active render effects (explosions, teleport folds).
- **Inputs:** `view` with active `effect` and `effect_phase`.
- **Outputs/Return:** None.
- **Side effects:** Modifies `world_to_screen_x/y` and `origin` fields; clears `effect` when phase expires.
- **Calls:** `shake_view_origin()`.
- **Notes:** Effect phase increments via `ticks_elapsed`; periodicity varies by effect type.

### instantiate_rectangle_transfer_mode
- **Signature:** `void instantiate_rectangle_transfer_mode(view_data *view, rectangle_definition *rectangle, short transfer_mode, _fixed transfer_phase)`
- **Purpose:** Apply sprite transfer mode (invisibility, static, fades, folds) to a rectangle definition.
- **Inputs:** `rectangle` to modify; `transfer_mode` enum; `transfer_phase` [0, FIXED_ONE]; `view` for shading mode.
- **Outputs/Return:** None.
- **Side effects:** Modifies `rectangle` fields: `transfer_mode`, `transfer_data`, `x0/x1`, `HorizScale`.
- **Calls:** `get_global_shading_table()`, `View_DoStaticEffect()`.
- **Notes:** Handles teleport in/out shrinking; infravision overrides invisibility.

### instantiate_polygon_transfer_mode
- **Signature:** `void instantiate_polygon_transfer_mode(struct view_data *view, struct polygon_definition *polygon, short transfer_mode, bool horizontal)`
- **Purpose:** Apply polygon transfer mode (slides, wanders, wobbles, pulsates) to floor/ceiling surfaces.
- **Inputs:** `polygon` to modify; `transfer_mode` enum; `horizontal` (floor vs. wall); `view` for tick timing.
- **Outputs/Return:** None.
- **Side effects:** Modifies `polygon` fields: `origin`, `vector`, `transfer_mode`, `transfer_data`.
- **Calls:** `sine_table[]`, `cosine_table[]`, `isqrt()`, `NORMALIZE_ANGLE()`.
- **Notes:** Animated slides/wanders use multi-frequency sine/cosine combinations; wobble scales with fast mode.

### render_viewer_sprite_layer
- **Signature:** `static void render_viewer_sprite_layer(view_data *view, RasterizerClass *RasPtr)`
- **Purpose:** Render the player's weapons and HUD elements as foreground sprites.
- **Inputs:** `view` ΓÇö camera state; `RasPtr` ΓÇö active rasterizer.
- **Outputs/Return:** None.
- **Side effects:** Calls rasterizer's `texture_rectangle()` multiple times.
- **Calls:** `get_weapon_display_information()`, `extended_get_shape_information()`, `position_sprite_axis()`, `extended_get_shape_bitmap_and_shading_table()`, `instantiate_rectangle_transfer_mode()`, `OGL_GetModelData()` (OpenGL).
- **Notes:** Loops through weapon frames; skips if `show_weapons_in_hand` is false; supports 3D model weapons via OpenGL.

### position_sprite_axis
- **Signature:** `static void position_sprite_axis(short *x0, short *x1, short scale_width, short screen_width, short positioning_mode, _fixed position, bool flip, world_distance world_left, world_distance world_right)`
- **Purpose:** Calculate screen coordinates for one axis of a sprite given positioning mode and world extents.
- **Inputs:** `positioning_mode` (low/center/high); `position` (fixed-point offset); `flip` (mirror); world dimensions.
- **Outputs/Return:** `x0, x1` ΓÇö calculated screen coordinates.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Handles three positioning modes; flipping inverts left/right world coordinates.

### shake_view_origin
- **Signature:** `static void shake_view_origin(struct view_data *view, world_distance delta)`
- **Purpose:** Apply earthquake shake effect to camera by offsetting origin with sine waves.
- **Inputs:** `view` ΓÇö camera state; `delta` ΓÇö shake magnitude.
- **Outputs/Return:** None.
- **Side effects:** Modifies `view->origin` if the new position remains in the same polygon.
- **Calls:** `sine_table[]`, `find_line_crossed_leaving_polygon()`.
- **Notes:** Uses three orthogonal frequency sine waves; only commits shake if polygon boundary not crossed.

## Control Flow Notes
**Frame rendering loop** (from `render_view`):
1. **Update**: `update_view_data()` adjusts camera vectors for frame.
2. **Visibility**: `RenderVisTree.build_render_tree()` traverses BSP/portal tree to mark visible polygons.
3. **Sorting**: `RenderSortPoly.sort_render_tree()` depth-sorts visible polygons and accumulates clipping.
4. **Object placement**: `RenderPlaceObjs.build_render_object_list()` finds and sorts visible objects (monsters, projectiles, scenery).
5. **Rasterization**: `RenderRasterize.render_tree()` clips and rasterizes each polygon and object (back-to-front).
6. **HUD**: `render_viewer_sprite_layer()` renders weapons over final scene.
7. **Overhead map** (optional): Drawn if enabled; **terminal mode** short-circuits to UI rendering.

This is a **frame/render phase** module, called once per game tick from the main game loop.

## External Dependencies
- **Notable includes:** map.h (geometry), render.h (declarations), interface.h (shapes/UI), lightsource.h, media.h, weapons.h, RenderVisTree.h, RenderSortPoly.h, RenderPlaceObjs.h, RenderRasterize.h (subsystem classes), Rasterizer_SW.h, Rasterizer_OGL.h, AnimatedTextures.h, OGL_Render.h.
- **Key external symbols:** `get_polygon_data()`, `get_endpoint_data()`, `get_light_intensity()`, `get_media_data()`, `get_weapon_display_information()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `OGL_IsActive()`, `OGL_GetModelData()`, `render_overhead_map()`, `render_computer_interface()` (defined elsewhere).
- **Trigonometry tables:** `sine_table[]`, `cosine_table[]` from world.h.
