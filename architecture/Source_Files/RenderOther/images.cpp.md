# Source_Files/RenderOther/images.cpp

## File Purpose
Manages loading and rendering of image resources (PICT pictures and CLUT color tables) from macOS resource forks and WAD files. Handles decompression of PackBits-RLE compressed picture data and provides full-screen picture rendering with optional scrolling.

## Core Responsibilities
- Load PICT, CLUT, sound, and text resources from resource files and WAD archives
- Decompress PackBits RLEΓÇôencoded picture data at depths 1/2/4/8/16/32-bit
- Convert macOS PICT resources to SDL_Surface for rendering
- Render full-screen pictures with keyboard/mouse-controlled scrolling
- Select appropriate picture resource IDs based on current display bit depth
- Convert WAD-format picture/CLUT data to macOS resource format (or vice versa)
- Manage dual image sources (main Images file + per-scenario scenario file)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `image_file_t` | class | Manages an opened image file (resource fork or WAD); owns resource and WAD file handles |
| `pict_head` | struct | PICT picture header (bounds, bit depth); macOS-only format |
| `clut_record` | struct | Color lookup table entry (count, ID, 256 RGB values); macOS-only format |
| `LoadedResource` | class | Wrapper for loaded resource data (pointer + length); auto-releases on destruction |
| `OpenedResourceFile` | class | Abstraction for resource fork file I/O |
| `OpenedFile` | class | Abstraction for WAD file I/O |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ImagesFile` | `image_file_t` | static | Primary image resource file (opened at startup) |
| `ScenarioFile` | `image_file_t` | static | Secondary image resource file for scenario/map (opened per-level) |

## Key Functions / Methods

### initialize_images_manager
- **Signature:** `void initialize_images_manager(void)`
- **Purpose:** Initialize image manager, open main Images file, and register shutdown handler.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Opens `ImagesFile`; registers `atexit()` handler for cleanup
- **Calls:** `FileSpecifier::SetNameWithPath()`, `ImagesFile.open_file()`, `alert_user()`
- **Notes:** Called during engine startup; fails fatally if Images file not found.

### set_scenario_images_file
- **Signature:** `void set_scenario_images_file(FileSpecifier &file)`
- **Purpose:** Load scenario-specific image resource file (overrides Images file for that level).
- **Inputs:** `file` ΓÇô scenario file specifier
- **Outputs/Return:** void
- **Side effects:** Opens `ScenarioFile`; subsequent image queries check this first
- **Calls:** `ScenarioFile.open_file()`

### picture_to_surface
- **Signature:** `SDL_Surface *picture_to_surface(LoadedResource &rsrc)`
- **Purpose:** Convert PICT resource to SDL_Surface.
- **Inputs:** `rsrc` ΓÇô PICT resource data (must be loaded)
- **Outputs/Return:** `SDL_Surface*` ΓÇô decoded surface, or `NULL` on failure
- **Side effects:** Allocates `SDL_Surface`; interprets PICT opcodes; decompresses pixel data
- **Calls:** `SDL_RWFromMem()`, `SDL_ReadBE16/32()`, `uncompress_picture()`, `SDL_CreateRGBSurface()`, `SDL_SetColors()`, `IMG_LoadTyped_RW()` (JPEG), `SDL_BlitSurface()`
- **Notes:** Parses PICT opcode stream (0x0098ΓÇô0x009b for CopyBits, 0x8200 for JPEG); handles 1/2/4/8/16/32-bit depths; keeps only first image encountered; supports banded JPEG data.

### uncompress_picture
- **Signature:** `static int uncompress_picture(const uint8 *src, int row_bytes, uint8 *dst, int dst_pitch, int depth, int height, int pack_type)`
- **Purpose:** Decompress PackBits RLE and perform color expansion (1/2/4ΓåÆ8-bit).
- **Inputs:** `src` ΓÇô compressed data; `row_bytes` ΓÇô logical width; `dst`, `dst_pitch` ΓÇô output buffer; `depth` ΓÇô bit depth; `height` ΓÇô rows; `pack_type` ΓÇô compression mode (0=uncompressed, 1=no packing, 3=16-bit RLE, 4=32-bit component RLE)
- **Outputs/Return:** int ΓÇô number of source bytes consumed
- **Side effects:** Writes decompressed pixels to `dst`; may malloc temporary buffer for color expansion
- **Calls:** `uncompress_rle8()`, `uncompress_rle16()`, `uncompress_rle32()`, `byte_swap_memory()`, `malloc()`, `free()`
- **Notes:** Allocates temp buffer if depth < 8 to expand colors before writing to dst.

### unpack_bits (template)
- **Signature:** `template <class T> static const uint8 *unpack_bits(const uint8 *src, int row_bytes, T *dst)`
- **Purpose:** Decompress single PICT scan line using PackBits RLE; handle endianness.
- **Inputs:** `src` ΓÇô RLE-encoded line; `row_bytes` ΓÇô expected uncompressed size; `dst` ΓÇô output buffer
- **Outputs/Return:** `const uint8*` ΓÇô pointer to byte after decompressed data
- **Side effects:** Writes decompressed scan line to `dst`
- **Calls:** None
- **Notes:** Handles variable-length count field (1 or 2 bytes); literal runs (c ΓëÑ 0) and RLE runs (c < 0).

### draw_picture (static)
- **Signature:** `static void draw_picture(LoadedResource &PictRsrc)`
- **Purpose:** Render PICT to screen (mostly visible in scroll version).
- **Inputs:** `PictRsrc` ΓÇô loaded PICT resource
- **Outputs/Return:** void
- **Side effects:** Blits surface to SDL_GetVideoSurface(); frees SDL_Surface
- **Calls:** `picture_to_surface()`, `rescale_surface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`

### scroll_full_screen_pict_resource_from_scenario
- **Signature:** `void scroll_full_screen_pict_resource_from_scenario(int pict_resource_number, bool text_block)` (partial signature inferred)
- **Purpose:** Render PICT with automatic scrolling and keyboard/mouse abort.
- **Inputs:** `pict_resource_number` ΓÇô resource ID; `text_block` ΓÇô affects scroll speed
- **Outputs/Return:** void
- **Side effects:** Renders animated scroll; processes SDL events; frees surface
- **Calls:** `get_picture_resource_from_scenario()`, `picture_to_surface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`, `SDL_GL_SwapBuffers()`, `global_idle_proc()`, `SDL_PollEvent()`
- **Notes:** Scrolls horizontally or vertically depending on picture size vs. 640├ù480 window; aborts on mouse/key press.

### get_picture_resource_from_images / get_picture_resource_from_scenario
- **Signature:** `bool get_picture_resource_from_images(int base_resource, LoadedResource &PictRsrc)`
- **Purpose:** Load PICT from Images or Scenario file, with fallback to lower bit depths.
- **Inputs:** `base_resource` ΓÇô resource ID; `PictRsrc` ΓÇô output resource
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Allocates resource data into `PictRsrc`
- **Calls:** `determine_pict_resource_id()`, `get_pict()`, `HNoPurge()` (Mac only)
- **Notes:** Scenario version checks file is open before attempting load.

### calculate_picture_clut
- **Signature:** `struct color_table *calculate_picture_clut(int CLUTSource, int pict_resource_number)`
- **Purpose:** Build color table from CLUT resource for 8-bit display.
- **Inputs:** `CLUTSource` ΓÇô `CLUTSource_Images` or `CLUTSource_Scenario`; `pict_resource_number` ΓÇô CLUT resource ID
- **Outputs/Return:** `color_table*` ΓÇô allocated table, or `NULL` if not found
- **Side effects:** Allocates `color_table`
- **Calls:** `OFilePtr->get_clut()`, `build_color_table()`, `build_direct_color_table()`
- **Notes:** Returns direct color table for 16/32-bit displays.

### image_file_t::open_file
- **Signature:** `bool image_file_t::open_file(FileSpecifier &file)`
- **Purpose:** Open image file as resource fork or WAD archive.
- **Inputs:** `file` ΓÇô file to open
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Opens `rsrc_file` and/or `wad_file`; reads WAD header if applicable
- **Calls:** `file.Open(rsrc_file)`, `open_wad_file_for_reading()`, `read_wad_header()`
- **Notes:** Tries resource fork first, falls back to WAD; tries to open both simultaneously.

### image_file_t::get_pict / get_clut / get_snd / get_text
- **Signature:** `bool image_file_t::get_pict(int id, LoadedResource &rsrc)`
- **Purpose:** Load resource from file (delegates to `get_rsrc` with appropriate type codes).
- **Inputs:** `id` ΓÇô resource ID; `rsrc` ΓÇô output resource
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Allocates resource data
- **Calls:** `get_rsrc()` with type/wad_type pairs

### image_file_t::get_rsrc
- **Signature:** `bool image_file_t::get_rsrc(uint32 rsrc_type, uint32 wad_type, int id, LoadedResource &rsrc)`
- **Purpose:** Load resource from resource fork or WAD file.
- **Inputs:** `rsrc_type` ΓÇô macOS type code (e.g., `'PICT'`); `wad_type` ΓÇô WAD type; `id` ΓÇô resource ID; `rsrc` ΓÇô output
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Allocates and sets resource data via `LoadedResource.SetData()`; converts PICT/CLUT from WAD format if needed
- **Calls:** `rsrc_file.Get()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `make_rsrc_from_pict()`, `make_rsrc_from_clut()`, `free_wad()`, `malloc()`, `memcpy()`
- **Notes:** Converts WAD PICT to resource format; extracts CLUT from WAD and passes to PICT conversion.

### image_file_t::determine_pict_resource_id
- **Signature:** `int image_file_t::determine_pict_resource_id(int base_id, int delta16, int delta32)`
- **Purpose:** Select appropriate PICT resource ID for current display bit depth.
- **Inputs:** `base_id` ΓÇô base ID (8-bit); `delta16` ΓÇô offset for 16-bit; `delta32` ΓÇô offset for 32-bit
- **Outputs/Return:** int ΓÇô actual resource ID to load
- **Side effects:** None
- **Calls:** `has_pict()`
- **Notes:** Tries 16/32-bit variants first, then 8-bit; returns 8-bit if none found.

### image_file_t::make_rsrc_from_pict (two versions: Mac + SDL)
- **Signature:** `bool image_file_t::make_rsrc_from_pict(void *data, size_t length, LoadedResource &rsrc, void *clut_data, size_t clut_length)`
- **Purpose:** Convert WAD PICT tag to macOS resource format.
- **Inputs:** `data` ΓÇô raw PICT data; `clut_data` ΓÇô optional CLUT for 8-bit; `rsrc` ΓÇô output
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Allocates PICT resource data; on Mac, creates GWorld and PICT handle
- **Calls:** (Mac) `QTNewGWorldFromPtr()`, `OpenPicture()`, `CopyBits()`, `ClosePicture()`, `malloc()`, `memcpy()`; (SDL) `malloc()`, `memset()`, `memcpy()`
- **Notes:** Mac version uses QuickTime; SDL version reconstructs PICT opcode stream from raw pixels.

### image_file_t::make_rsrc_from_clut
- **Signature:** `bool image_file_t::make_rsrc_from_clut(void *data, size_t length, LoadedResource &rsrc)`
- **Purpose:** Convert WAD CLUT tag (6 bytes header + 256├ù6 RGB entries) to macOS format (8 bytes header + 256├ù8 entries).
- **Inputs:** `data` ΓÇô raw CLUT; `rsrc` ΓÇô output
- **Outputs/Return:** bool ΓÇô success
- **Side effects:** Allocates CLUT resource
- **Calls:** `malloc()`, `memset()`, `LoadedResource.SetData()`

## Control Flow Notes
**Initialization phase:** `initialize_images_manager()` opens ImagesFile at engine startup, registers shutdown handler.  
**Level load phase:** `set_scenario_images_file()` opens scenario-specific image file.  
**UI/Render phase:** `draw_full_screen_pict_resource_from_images/scenario()` and `scroll_full_screen_pict_resource_from_scenario()` render pictures for menus, intro screens, cutscenes.  
**Shutdown phase:** `shutdown_images_handler()` closes both files.

## External Dependencies
- **SDL/SDL_image:** `SDL_RWops`, `SDL_Surface`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_ReadBE16/32()`, `IMG_LoadTyped_RW()` (JPEG)
- **FileHandler.h:** `OpenedResourceFile`, `OpenedFile`, `LoadedResource`, `FileSpecifier`
- **wad.h:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `extract_type_from_wad()`, `free_wad()`
- **shell.h:** `global_idle_proc()`, `alert_user()`
- **screen.h/screen_drawing.h:** `interface_bit_depth` (external), `draw_clip_rect_active`, `draw_clip_rect`
- **byte_swapping.h:** `byte_swap_memory()`
- **cseries.h:** Base types, `FOUR_CHARS_TO_INT()`, `assert()`, `MIN()`
- **MacOS QuickTime** (conditional `#ifdef mac`): `QTNewGWorldFromPtr()`, `OpenPicture()`, `ClosePicture()`, `CopyBits()`, graphics state management
