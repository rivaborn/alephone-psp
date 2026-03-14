# Source_Files/RenderMain/shapes.cpp

## File Purpose
Manages game shape (sprite/texture) collections including loading, rendering, and visual effects. Handles shape data parsing, bitmap unpacking, shading table generation, and SDL surface creation for 2D rendering with support for RLE-encoded shapes and dynamic illumination.

## Core Responsibilities
- Load and cache shape collections from binary resource files
- Unpack RLE-encoded bitmap data and create SDL surfaces
- Build color shading tables for darkness/lighting effects and infravision
- Manage collection lifecycle (load, strip, unload) with memory pooling
- Apply per-pixel illumination, mirroring, and shrinking transformations
- Parse and apply infravision tint configuration via XML
- Provide accessors for high-level, low-level, and bitmap shape definitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_definition` | struct | Holds color tables (CLUTs), high/low-level shapes, and bitmaps for one collection |
| `collection_header` | struct (external) | Runtime state for a loaded collection (shading tables, tint tables, status flags) |
| `bitmap_definition` | struct | Raw bitmap metadata (width, height, bytes-per-row, RLE flag) and pixel data pointers |
| `high_level_shape_definition` | struct | Animation frame metadata (views, frames-per-view, ticks, sound cues, low-level shape indices) |
| `low_level_shape_definition` | struct | Single-frame shape (bitmap index, world bounds, origin/key points, mirror flags) |
| `rgb_color_value` | struct | Color with flags (self-luminescent bit), value index, and 16-bit RGBA components |
| `OpenedFile` | class (external) | Platform-agnostic file handle with read/seek/length operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `global_shading_table16` | `pixel16*` | static | 16-bit pre-computed shading lookup table (all collections, all levels) |
| `global_shading_table32` | `pixel32*` | static | 32-bit pre-computed shading lookup table (all collections, all levels) |
| `number_of_shading_tables` | `short` | global | Count of darkness levels (e.g., 32 or 64) |
| `shading_table_size` | `short` | global | Bytes per darkness level per CLUT |
| `ShapesFile` | `OpenedFile` | static | Currently open shapes resource file |
| `CollectionTints[NUMBER_OF_COLLECTIONS]` | `short[]` | static | Per-collection infravision tint color assignments |
| `original_tint_colors16` | `rgb_color*` | static | Backup of tint color array (for XML reset) |
| `OriginalCollectionTints` | `short*` | static | Backup of collection tints (for XML reset) |

## Key Functions / Methods

### get_shape_surface
- **Signature:** `SDL_Surface *get_shape_surface(int shape, int inCollection, byte** outPointerToPixelData, float inIllumination, bool inShrinkImage)`
- **Purpose:** Convert a shape descriptor to an SDL_Surface for rendering, with optional illumination-based coloring and image shrinking.
- **Inputs:** 
  - `shape`: shape descriptor (encodes collection, CLUT, low-level index) or low-level index if `inCollection != NONE`
  - `inCollection`: override collection spec, or NONE to use descriptor
  - `outPointerToPixelData`: optional output for malloc'd pixel buffer (RLE shapes only)
  - `inIllumination`: [0.0ΓÇô1.0] to use shading table colors; <0 uses CLUT
  - `inShrinkImage`: halve dimensions via nearest-neighbor (RLE only)
- **Outputs/Return:** SDL_Surface with 8-bit indexed color + palette; NULL on error. Pixel data ownership transferred to caller if `outPointerToPixelData` is non-NULL.
- **Side effects:** Allocates SDL surface and pixel buffer; calls SDL_SetColors and SDL_SetColorKey.
- **Calls:** `get_collection_definition`, `get_low_level_shape_definition`, `extended_get_shape_bitmap_and_shading_table`, `get_bitmap_definition`, `get_collection_colors`, `SDL_CreateRGBSurfaceFrom`, RLE unpacking loop.
- **Notes:** Handles both row-major and column-major RLE with X/Y mirroring. Shrink only works for RLE. Caller must free() pixel data before freeing surface. Self-luminescent colors skip darkening in illumination mode.

### load_collection
- **Signature:** `static bool load_collection(short collection_index, bool strip)`
- **Purpose:** Read and parse a collection resource from the shapes file into memory.
- **Inputs:**
  - `collection_index`: which collection to load (0ΓÇô31)
  - `strip`: if true, load shape metadata only (skip bitmaps)
- **Outputs/Return:** true on success; sets collection header status.
- **Side effects:** Allocates collection_definition and shading tables; sets collection_header->collection and ->shading_tables. Calls PSP profiler marks.
- **Calls:** `get_collection_header`, `load_collection_definition`, `load_clut`, `load_high_level_shape`, `load_low_level_shape`, `load_bitmap`, `allocate_shading_tables`, `build_collection_tinting_table`, `byte_swap_memory`, SDL_RWops read functions.
- **Notes:** Uses `std::auto_ptr<collection_definition>` for exception safety. Detects bit_depth (8 vs 16) and reads offset16 if needed. Invokes progress callbacks. Profile tags key subsections.

### build_shading_tables16
- **Signature:** `static void build_shading_tables16(rgb_color_value *colors, short color_count, pixel16 *shading_tables, byte *remapping_table, bool is_opengl)`
- **Purpose:** Populate a 16-bit shading table (one level per darkness) from a CLUT, with self-luminescence support.
- **Inputs:**
  - `colors`: color palette (may include remapping)
  - `color_count`: size of palette
  - `shading_tables`: output buffer (PIXEL8_MAXIMUM_COLORS ├ù number_of_shading_tables entries)
  - `remapping_table`: optional color index permutation; NULL to use identity
  - `is_opengl`: if true, use Mac xRGB 1555 format; if false, use SDL_MapRGB
- **Outputs/Return:** (none; modifies shading_tables in place)
- **Side effects:** Writes 16-bit pixel values to shading_tables; may call SDL_MapRGB if SDL and !is_opengl.
- **Calls:** `objlist_set`, `get_next_color_run`, `SDL_MapRGB` (conditional).
- **Notes:** Iterates color runs (contiguous colors with same brightness trend). Self-luminescent colors (flagged SELF_LUMINESCENT_COLOR_FLAG) use mid-to-bright range even at low darkness levels.

### build_global_shading_table16
- **Signature:** `static void build_global_shading_table16(void)`
- **Purpose:** Pre-compute component-wise shading for all darkness levels and RGB components (for fast integer multiplication).
- **Inputs:** (none; uses global `number_of_shading_tables`)
- **Outputs/Return:** (none; allocates and populates global_shading_table16)
- **Side effects:** malloc's ~32KΓÇô128K (platform-dependent bit depth); asserts non-NULL.
- **Calls:** (none visible; raw allocation and loops)
- **Notes:** Stores pre-scaled R, G, B values for each shading level, accounting for SDL pixel format component widths/shifts. MacOS assumed xRGB 1555 (5 bits per component). Only called once (guarded by !global_shading_table16 check).

### Accessor Functions
- **get_collection_definition(collection_index)**: Returns `collection_header->collection` or NULL if out-of-range.
- **get_collection_colors(index, clut_num)**: Returns pointer to color table in collection; NULL if invalid CLUT index.
- **get_low_level_shape_definition(index, shape_idx)**: Returns low-level shape struct; NULL if shape_idx out-of-bounds.
- **get_bitmap_definition(index, bitmap_idx)**: Returns bitmap definition cast from vector storage.
- **get_collection_shading_tables(index, clut_idx)**: Returns pointer into collection header's shading table buffer, offset by CLUT.
- **get_collection_tint_tables(index, tint_idx)**: Returns pointer to tint table (stored after shading tables in same buffer).

### XML Infravision Configuration
- **XML_InfravisionAssignParser (class)**: Parses `<assign coll="X" color="Y"/>` elements to override per-collection tint colors. Backs up originals on first Start().
- **XML_InfravisionParser (class)**: Parent parser for `<infravision>` block; delegates color parsing to Color_GetParser().
- **Infravision_GetParser()**: Returns parser chain for XML config; attaches assign and color parsers as children.

## Control Flow Notes
This module integrates into the render initialization and level-load pipeline:
1. **Init:** `initialize_shape_handler()` is called early (defined elsewhere; not shown but referenced in shell.h).
2. **Level Load:** Collections are marked for loading via `mark_collection()` (defined elsewhere). `load_collections()` (defined elsewhere) iterates and calls `load_collection()` for marked indices.
3. **Render Frame:** Shapes are looked up via accessors and converted to SDL surfaces on-demand via `get_shape_surface()`.
4. **Shutdown:** `shutdown_shape_handler()` and `close_shapes_file()` (both defined elsewhere or stubbed here) clean up resources.

XML infravision config is loaded during map preprocessing, overriding static `CollectionTints[]` assignments.

## External Dependencies
- **SDL (audio/graphics):** SDL_RWops, SDL_ReadBE16/32, SDL_MapRGB, SDL_CreateRGBSurfaceFrom, SDL_SetColors, SDL_SetColorKey, SDL_GetVideoSurface, SDL_PixelFormat, SDL_SwapBE16, SDL_CreateRGBSurfaceFrom.
- **Collection metadata:** collection_definition.h defines struct layouts; collection_header is declared in shape_definitions.h (not shown).
- **File I/O:** FileHandler.h (OpenedFile class); shapes file opened via open_shapes_file() (defined elsewhere).
- **OpenGL:** OGL_Render.h, OGL_LoadScreen.h; OGL_SetInfravisionTint() called conditionally.
- **Color parsing:** ColorParser.h; Color_GetParser() and Color_SetArray() used for XML.
- **Byte order:** byte_swapping.h; byte_swap_memory() for endian conversion.
- **Texture extras:** SW_Texture_Extras.h, Packing.h (headers included; purpose unclear from this file).
- **XML parsing:** XML_ElementParser base class; cseries.h and interface.h for macros and assertions.
- **Profiling:** psp_sdl_profilermg.h for PSP target (PSPROF_* macros).
