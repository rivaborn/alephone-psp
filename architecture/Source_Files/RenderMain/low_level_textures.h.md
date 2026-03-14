# Source_Files/RenderMain/low_level_textures.h

## File Purpose
Low-level templated texture rasterization for the Marathon-like game engine. Provides functions to render horizontal and vertical textured polygon scanlines directly to a bitmap surface, with support for multiple pixel depths (8/16/32-bit), alpha blending modes, transparency, tinting, and dithering.

## Core Responsibilities
- Pixel blending primitives: averaging and alpha-blending for different bit depths
- Horizontal scanline texture mapping with per-pixel source address calculation
- Optimized landscape texture rendering with fixed downshift calculations
- Vertical column texture mapping with 4-pixel-wide SIMD-style optimization
- Transparency-aware pixel writes with configurable alpha blend modes
- Color tinting operations via lookup tables
- Stochastic dithering/randomization effects on textured surfaces
- Support for per-pixel opacity tables in alpha blending

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `bitmap_definition` | struct | Screen/texture bitmap with row addressing (defined elsewhere) |
| `_horizontal_polygon_line_data` | struct | Per-scanline texture source coordinates and shading table |
| `_vertical_polygon_data` | struct | Vertical column metadata: width, x offset, downshift factor |
| `_vertical_polygon_line_data` | struct | Per-column texture source, shading table, and tint table pointers |
| `tint_table8` / `tint_table16` / `tint_table32` | struct | Color component lookup tables for tinting (defined elsewhere) |
| `SDL_PixelFormat` | struct | SDL pixel format info: masks, shifts, loss factors |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `world_pixels` | extern `SDL_Surface*` | global | Main screen/framebuffer surface; accessed for pixel format info and row addressing |
| `texture_random_seed` | extern `uint16` | global | RNG seed for dithering in `randomize_vertical_polygon_lines()` |
| `number_of_shading_tables` | extern int | global | Bounds-check for tint table indices in assertions |

## Key Functions / Methods

### average (generic + specializations)
- Signature: `template <typename T> inline T average(T fg, T bg)` + `pixel16` / `pixel32` specializations
- Purpose: Blend two pixels via 50% average (fast blending mode)
- Inputs: foreground pixel, background pixel
- Outputs/Return: blended pixel
- Side effects: none
- Calls: none (inline bitwise ops)
- Notes: Generic returns foreground (identity); `pixel16` assumes 565 format; `pixel32` assumes ARGB with avoid-overflow bit math

### alpha_blend
- Signature: `template <typename T> inline T alpha_blend(T fg, T bg, pixel8 alpha, pixel32 rmask, pixel32 gmask, pixel32 bmask)`
- Purpose: Per-channel alpha blend with color masks
- Inputs: foreground, background, alpha (0ΓÇô255), RGB component masks
- Outputs/Return: blended pixel
- Side effects: none
- Calls: none
- Notes: Scales foreground-background delta by alpha per channel; relies on caller providing correct masks for pixel type

### write_pixel
- Signature: `template <typename T, int sw_alpha_blend, bool check_transparency> void inline write_pixel(...)`
- Purpose: Write a single textured pixel to destination with optional transparency check and alpha blending
- Inputs: destination pointer, palette index, shading table, opacity table, color masks
- Outputs/Return: modifies destination pixel in place
- Side effects: writes to framebuffer
- Calls: `average()`, `alpha_blend()` (conditionally)
- Notes: `check_transparency` skips write if palette index is 0; `sw_alpha_blend` enum selects blend mode

### texture_horizontal_polygon_lines
- Signature: `template <typename T, int sw_alpha_blend> void texture_horizontal_polygon_lines(...)`
- Purpose: Rasterize a horizontal scanline span with bilinear texture mapping
- Inputs: texture bitmap, screen bitmap, scanline data array, y-offset, x-start/end tables, line count, optional opacity table
- Outputs/Return: pixels written to screen
- Side effects: writes to framebuffer; reads `world_pixels->format` if using nice alpha blend
- Calls: `write_pixel()` per destination pixel
- Notes: Uses fixed-point source coordinates; polls texture at `(source_y >> shift, source_x >> shift)` per pixel; increments source by `source_dx`, `source_dy`

### landscape_horizontal_polygon_lines
- Signature: `template <typename T> void landscape_horizontal_polygon_lines(...)`
- Purpose: Optimized horizontal scanline rendering for landscape/terrain textures
- Inputs: texture (assumed square power-of-2), screen bitmap, scanline data, y-offset, x-start/end tables, line count
- Outputs/Return: pixels written to screen
- Side effects: writes to framebuffer
- Calls: none (inline shading table lookup)
- Notes: Uses `NextLowerExponent(texture->height)` to compute x-downshift dynamically; faster than general case because no y-coordinate interpolation

### texture_vertical_polygon_lines
- Signature: `template <typename T, int sw_alpha_blend, bool check_transparent> void texture_vertical_polygon_lines(...)`
- Purpose: Rasterize vertical column spans with per-pixel alpha blending and transparency
- Inputs: screen bitmap, view data, vertical polygon data, y-start/end tables, optional opacity table
- Outputs/Return: pixels written to screen
- Side effects: writes to framebuffer; reads `world_pixels->format` if using nice alpha blend
- Calls: `write_pixel()`, `copy_check_transparent()` (conditional on line_count >= 4)
- Notes: Optimized path processes 4 columns in lockstep if aligned and sufficient height; falls back to single-column path for edges. Detailed sync/parallel/desync phases to handle misaligned column heights.

### copy_check_transparent
- Signature: `template <typename T, bool check_transparent> void inline copy_check_transparent(T *dst, pixel8 read, T *shading_table)`
- Purpose: Write a pixel if non-transparent (helper for vertical rendering)
- Inputs: destination pointer, palette index, shading table
- Outputs/Return: modifies destination pixel
- Side effects: writes to framebuffer
- Calls: none
- Notes: Used in "sync" and "desync" phases of vertical renderer to handle partial columns

### tint_vertical_polygon_lines
- Signature: `template <typename T> void tint_vertical_polygon_lines(..., uint16 transfer_data)`
- Purpose: Apply color tinting to existing screen pixels in a vertical column span
- Inputs: screen bitmap, view data, vertical polygon data, y-tables, tint table index
- Outputs/Return: pixels modified in place on screen
- Side effects: writes to framebuffer; reads `world_pixels->format`
- Calls: `tint_tables_pointer()`, `get_pixel_tint()` per destination pixel
- Notes: Only modifies pixels where texture is non-transparent; uses lookup tables for fast color remapping

### randomize_vertical_polygon_lines
- Signature: `template <typename T> void randomize_vertical_polygon_lines(..., uint16 transfer_data)`
- Purpose: Apply stochastic dithering/noise to vertical column spans
- Inputs: screen bitmap, view data, vertical polygon data, y-tables, drop threshold (transfer_data)
- Outputs/Return: pixels randomly modified on screen
- Side effects: writes to framebuffer; updates global `texture_random_seed` LFSR state
- Calls: `randomize_vertical_polygon_lines_write()` (conditional)
- Notes: Only affects non-transparent texels; uses Fibonacci LFSR (tap 0xb400) for deterministic PRN; output is sparse if threshold is high

### tint_tables_pointer / get_pixel_tint / randomize_vertical_polygon_lines_write
- Generic + 3 specializations each (pixel8, pixel16, pixel32)
- Provide polymorphic access to pixel format-specific lookup tables and operations
- Notes: Generic versions return 0/identity; specializations cast/reinterpret shading table pointers and extract/reconstruct color components

## Control Flow Notes
These functions are called during the frame rendering pipeline, likely from a higher-level polygon rasterizer that walks edge tables. **Horizontal functions** iterate scanlines top-to-bottom; **vertical functions** iterate columns left-to-right, with the vertical path optimized for 4-column SIMD-like processing when memory-aligned. All functions assume caller has pre-computed shading and opacity tables, and that bitmap row pointers are valid.

## External Dependencies
- **SDL library**: `SDL_Surface`, `SDL_PixelFormat` (pixel format metadata, framebuffer access)
- **Macro**: `FIXED_INTEGERAL_PART()` ΓÇö integer part of a fixed-point value (defined elsewhere)
- **Function**: `NextLowerExponent()` ΓÇö logΓéé of next lower power of 2 (defined elsewhere)
- **Types**: `bitmap_definition`, `view_data`, `_horizontal_polygon_line_data`, `_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table*` (all defined elsewhere in engine headers)
- **Global externs**: `world_pixels`, `texture_random_seed`, `number_of_shading_tables`
