# Source_Files/RenderMain/scottish_textures.cpp

## File Purpose
Software texture rasterizer for the Marathon/Aleph One game engine. Maps 2D textures onto screen-space polygons and rectangles using precalculated coordinate tables and Bresenham-style line algorithms, with support for multiple color depths, alpha blending, and landscape textures.

## Core Responsibilities
- Texture-map convex polygons (both horizontal and vertical scan-line orientations)
- Texture-map axis-aligned rectangles with perspective correction
- Landscape/terrain texture rendering with tiling and repeat options
- Precalculate texture coordinates and shading tables before rasterization
- Build Bresenham line-drawing tables for polygon edge traversal
- Dispatch to templated low-level pixel writers (8/16/32-bit, with/without alpha)
- Handle multiple transfer modes (textured, static/randomized, tinted, landscaped)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `_horizontal_polygon_line_header` | struct | Per-line metadata for horizontal polygon rendering (Y downshift factor) |
| `_horizontal_polygon_line_data` | struct | Precalculated texture source coordinates and deltas for one scan line |
| `_vertical_polygon_data` | struct | Column-level header: downshift, starting X, column width |
| `_vertical_polygon_line_data` | struct | Per-column texture Y coordinate, Y delta, texture row pointer, and shading table |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scratch_table0` | `short*` | static | Scratch buffer for left-edge X/Y coordinate tables |
| `scratch_table1` | `short*` | static | Scratch buffer for right-edge X/Y coordinate tables |
| `precalculation_table` | `void*` | static | Precalculated per-line texture data (headers + line data) |
| `texture_random_seed` | `uint16` | static | RNG seed for static-transfer-mode pixel randomization |

## Key Functions / Methods

### allocate_texture_tables
- **Signature:** `void allocate_texture_tables(void)`
- **Purpose:** Initialize global scratch and precalculation buffers at engine startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates 2├ù `short[MAXIMUM_SCRATCH_TABLE_ENTRIES]` and 1├ù precalculation table via `new[]`.
- **Calls:** `new[]`
- **Notes:** Called once during init. Asserts allocation success.

### texture_horizontal_polygon
- **Signature:** `void Rasterizer_SW_Class::texture_horizontal_polygon(polygon_definition& textured_polygon)`
- **Purpose:** Rasterize a textured polygon by scanning left-to-right, precalculating texture data then dispatching to low-level renderers.
- **Inputs:** `polygon_definition` with vertices, texture, transfer mode, shading tables, depth, flags.
- **Outputs/Return:** Draws pixels to screen buffer (side effect).
- **Side effects:** Fills `scratch_table0/1` with edge X coordinates, fills `precalculation_table` with line texture data, writes to screen framebuffer.
- **Calls:** `build_x_table()`, `_pretexture_horizontal_polygon_lines()`, `_prelandscape_horizontal_polygon_lines()`, templated `texture_horizontal_polygon_lines<>()` and `landscape_horizontal_polygon_lines<>()` from `low_level_textures.h`, `SW_Texture_Extras::instance()->GetTexture()`.
- **Notes:** Defers to `texture_vertical_polygon()` if transfer mode is `_static_transfer`. Uses nested switch on bit-depth (8/16/32) and transfer mode. For 16/32-bit, checks `graphics_preferences->software_alpha_blending` to select alpha blend variant.

### texture_vertical_polygon
- **Signature:** `void Rasterizer_SW_Class::texture_vertical_polygon(polygon_definition& textured_polygon)`
- **Purpose:** Rasterize a textured polygon by scanning top-to-bottom, precalculating texture data then dispatching to low-level renderers.
- **Inputs:** `polygon_definition` with vertices, texture, transfer mode, shading tables, depth, flags.
- **Outputs/Return:** Draws pixels to screen buffer (side effect).
- **Side effects:** Fills `scratch_table0/1` with edge Y coordinates, fills `precalculation_table` with column texture data, writes to screen framebuffer.
- **Calls:** `build_y_table()`, `_pretexture_vertical_polygon_lines()`, templated `texture_vertical_polygon_lines<>()` from `low_level_textures.h`, `SW_Texture_Extras::instance()->GetTexture()`.
- **Notes:** Defers to `texture_horizontal_polygon()` if transfer mode is `_big_landscaped_transfer`. Supports `_textured_transfer` and `_static_transfer` modes only in vertical path. Transparency handled via polygon texture flags.

### texture_rectangle
- **Signature:** `void Rasterizer_SW_Class::texture_rectangle(rectangle_definition& textured_rectangle)`
- **Purpose:** Rasterize a textured rectangle (axis-aligned, with clipping), handling per-column texture offset calculations.
- **Inputs:** `rectangle_definition` with bounds, texture, transfer mode, clipping, flip/mirroring, shading tables, depth.
- **Outputs/Return:** Draws pixels to screen buffer (side effect).
- **Side effects:** Clips to screen and rectangle bounds, fills `scratch_table0/1` with Y coordinate tables, fills `precalculation_table` with column data, writes to screen framebuffer.
- **Calls:** `calculate_shading_table()`, templated `texture_vertical_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()` from `low_level_textures.h`.
- **Notes:** Handles horizontal flipping. Reads texture row headers (first/last valid Y) for vertical clipping. Supports `_textured_transfer`, `_static_transfer`, `_tinted_transfer`.

### _pretexture_horizontal_polygon_lines
- **Signature:** `static void _pretexture_horizontal_polygon_lines(polygon_definition*, bitmap_definition*, view_data*, _horizontal_polygon_line_data*, short y0, short* x0_table, short* x1_table, short line_count)`
- **Purpose:** Precalculate texture source coordinates and deltas for each scan line of a horizontal polygon.
- **Inputs:** Polygon definition, view parameters, edge X tables, starting Y, line count.
- **Outputs/Return:** Fills precalculation table with texture origin/delta per line.
- **Side effects:** Fills `precalculation_table`.
- **Calls:** `calculate_shading_table()`, trigonometric tables (`cosine_table`, `sine_table`).
- **Notes:** Uses world-to-screen perspective transforms. Distinguishes high-precision path when polygon origin.z is in `[-WORLD_ONE, WORLD_ONE]`.

### _pretexture_vertical_polygon_lines
- **Signature:** `static void _pretexture_vertical_polygon_lines(polygon_definition*, bitmap_definition*, view_data*, _vertical_polygon_data*, short x0, short* y0_table, short* y1_table, short line_count)`
- **Purpose:** Precalculate texture coordinates and deltas for each column of a vertical polygon.
- **Inputs:** Polygon definition, view parameters, edge Y tables, starting X, line count.
- **Outputs/Return:** Fills precalculation table (header + per-column line data).
- **Side effects:** Fills `precalculation_table`.
- **Calls:** `calculate_shading_table()`.
- **Notes:** Performs fixed-point perspective division to map screen columns to texture columns. Includes overflow checks and rescaling to avoid integer overflow in numerator/denominator.

### _prelandscape_horizontal_polygon_lines
- **Signature:** `static void _prelandscape_horizontal_polygon_lines(polygon_definition*, bitmap_definition*, view_data*, _horizontal_polygon_line_data*, short y0, short* x0_table, short* x1_table, short line_count)`
- **Purpose:** Precalculate landscape texture coordinates (repeating sky/terrain textures) for scan lines.
- **Inputs:** Polygon definition, view parameters, edge X tables, starting Y, line count.
- **Outputs/Return:** Fills precalculation table with landscape texture source X and Y per line.
- **Side effects:** Fills `precalculation_table`.
- **Calls:** `View_GetLandscapeOptions()`, `calculate_shading_table()`, `NextLowerExponent()`.
- **Notes:** Applies landscape-specific yaw and tiling options (e.g., vertical repeat). Uses horizontal/vertical pixel deltas from view cone half-angle.

### build_x_table
- **Signature:** `static short* build_x_table(short* table, short x0, short y0, short x1, short y1)`
- **Purpose:** Bresenham-style line algorithm: build a table of X coordinates for every Y value from y0 to y1.
- **Inputs:** Output table, start (x0, y0), end (x1, y1).
- **Outputs/Return:** Pointer to table (or NULL if dy Γëñ 0).
- **Side effects:** Fills table with X values.
- **Calls:** None.
- **Notes:** Requires dy > 0. Handles X-dominant and Y-dominant cases. Returns NULL for non-positive dy (safety check).

### build_y_table
- **Signature:** `static short* build_y_table(short* table, short x0, short y0, short x1, short y1)`
- **Purpose:** Bresenham-style line algorithm: build a table of Y coordinates for every X value from x0 to x1.
- **Inputs:** Output table, start (x0, y0), end (x1, y1).
- **Outputs/Return:** Pointer to table (or NULL if dx < 0).
- **Side effects:** Fills table with Y values.
- **Calls:** None.
- **Notes:** Requires dx ΓëÑ 0. Handles X-dominant and Y-dominant cases. Supports backward writing (dy < 0) by using descending pointers.

### calculate_shading_table
- **Signature:** `static void calculate_shading_table(void*& result, view_data*, void* shading_tables, short depth, _fixed ambient_shade)`
- **Purpose:** Select a shading/lighting lookup table based on depth and ambient light.
- **Inputs:** View data, shading table array, depth distance, ambient shade intensity.
- **Outputs/Return:** Fills `result` with pointer to appropriate shading table for the given bit depth.
- **Side effects:** None (pure lookup).
- **Calls:** Macro `SHADE_TO_SHADING_TABLE_INDEX()`, `DEPTH_TO_SHADE()`, global `bit_depth`, `number_of_shading_tables`.
- **Notes:** Uses maximum depth intensity from view to compute final shade. Clamps to [0, FIXED_ONE]. Blends ambient and depth-based shade. Selects table offset based on pixel byte width (8/16/32-bit).

### NextLowerExponent
- **Signature:** `inline int NextLowerExponent(int n)`
- **Purpose:** Find the exponent k such that 2^k Γëñ n < 2^(k+1).
- **Inputs:** Integer n > 0.
- **Outputs/Return:** Exponent (e.g., NextLowerExponent(128) ΓåÆ 7, NextLowerExponent(256) ΓåÆ 8).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Used to compute landscape texture width bits from height dimension.

**Notes (helper functions):**
- Trivial helpers not documented separately: Macro `SHADE_TO_SHADING_TABLE_INDEX()` converts fixed-point shade to table index; `DEPTH_TO_SHADE()` converts world distance to shade value; `MIN()`, `MAX()`, `PIN()`, `ABS()`, `SGN()` are standard math utilities.

## Control Flow Notes
- **Initialization phase:** `allocate_texture_tables()` allocates scratch buffers once.
- **Render phase (per polygon/rectangle):** Main entry points are called (`texture_horizontal_polygon()`, `texture_vertical_polygon()`, `texture_rectangle()`).
  - Polygon functions: traverse edges with Bresenham, precalculate texture data, dispatch to templated low-level renderers in `low_level_textures.h`.
  - Rectangle function: clip to screen/bounds, precalculate per-column texture offsets, dispatch to vertical polygon renderer.
- **Templated dispatch:** Low-level rendering is deferred to header-only templates (in `low_level_textures.h`) to generate code for all combinations of pixel type (pixel8/16/32) and alpha blend mode.

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö Core types, assertions, utilities
  - `render.h` ΓÇö `polygon_definition`, `rectangle_definition`, `view_data`, `bitmap_definition`, render flags
  - `Rasterizer_SW.h` ΓÇö `Rasterizer_SW_Class` method container
  - `preferences.h` ΓÇö `graphics_preferences` global (for alpha blending options)
  - `SW_Texture_Extras.h` ΓÇö `SW_Texture_Extras` singleton for per-shape alpha tables
  - `low_level_textures.h` ΓÇö Templated pixel writing functions (`texture_horizontal_polygon_lines<>()`, `texture_vertical_polygon_lines<>()`, `landscape_horizontal_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()`)

- **External globals used:**
  - `bit_depth` ΓÇö Current pixel color depth (8, 16, or 32 bits)
  - `graphics_preferences` ΓÇö Rendering preferences (software alpha blending mode)
  - `number_of_shading_tables` ΓÇö Count of lighting lookup tables
  - `cosine_table[]`, `sine_table[]` ΓÇö Trigonometric lookup tables
  - `FIXED_FRACTIONAL_BITS`, `WORLD_FRACTIONAL_BITS`, `ANGULAR_BITS`, `TRIG_SHIFT` ΓÇö Fixed-point precision constants

- **External functions called:**
  - `View_GetLandscapeOptions(shape_descriptor)` ΓÇö Retrieve landscape tiling parameters (defined elsewhere)
  - `SW_Texture_Extras::instance()->GetTexture(shape_descriptor)` ΓÇö Retrieve per-shape opacity table (if alpha blending enabled)
