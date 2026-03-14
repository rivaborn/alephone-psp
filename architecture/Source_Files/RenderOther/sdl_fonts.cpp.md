# Source_Files/RenderOther/sdl_fonts.cpp

## File Purpose
Manages font loading, caching, and text metrics for the Aleph One game engine. Supports both legacy bitmap fonts (from Mac resources) and modern TrueType fonts with style variants (bold, italic, etc.).

## Core Responsibilities
- Initialize font system by scanning data directories for font resources
- Load and cache bitmap fonts from FOND/NFNT/FONT resource structures
- Load and cache TrueType fonts with automatic fallback chains for style variants
- Convert bitmap font data from 1-bit-per-pixel to 1-byte-per-pixel pixmaps
- Calculate text width for both font types, handling style codes and shadows
- Truncate text to fit within max width
- Parse and apply inline style codes (|b, |i, |p) in text strings
- Manage reference counting for cached fonts

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sdl_font_info` | struct | Bitmap font metadata: pixmap, char tables, dimensions, ref count |
| `ttf_font_info` | struct | TrueType font: array of TTF_Font* for style variants, height adjustment |
| `font_info` | base class | Abstract interface for both bitmap and TTF fonts |
| `id_and_size_t` | typedef | Key for bitmap font cache: (font_id, size) |
| `ttf_font_key_t` | typedef | Key for TTF cache: (file_path, style, size) |
| `style_separator` | class | Boost tokenizer for parsing "|b", "|i" style codes in text |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `font_list` | `map<id_and_size_t, sdl_font_info*>` | static | Cache of loaded bitmap fonts |
| `ttf_font_list` | `map<ttf_font_key_t, ref_counted_ttf_font_t>` | static | Cache of loaded TTF fonts with ref counts |
| `data_search_path` | `vector<DirectorySpecifier>` | extern | Data directories (from shell_sdl.cpp) |

## Key Functions / Methods

### initialize_fonts
- **Signature:** `void initialize_fonts(void)`
- **Purpose:** Initialize font subsystem at startup; open font resource files or fallback to built-in TTF.
- **Inputs:** None (reads `data_search_path` externally)
- **Outputs/Return:** None; calls `exit(1)` on fatal failure
- **Side effects:** Opens resource files with `open_res_file()`, may call `fix_missing_overhead_map_fonts()` and `fix_missing_interface_fonts()`
- **Calls:** `open_res_file()`, `logContext()`, `logFatal()`, `exit()`
- **Notes:** Searches for "Fonts" or "Fonts.fntA" in data directories; fatal if neither found and SDL_TTF disabled.

### load_sdl_font
- **Signature:** `sdl_font_info *load_sdl_font(const TextSpec &spec)`
- **Purpose:** Load a bitmap font by ID and size; return cached entry or allocate new.
- **Inputs:** `TextSpec` with `font` (resource ID) and `size`
- **Outputs/Return:** Pointer to `sdl_font_info` or NULL on error
- **Side effects:** Allocates `sdl_font_info` and pixmap buffer; modifies `font_list` cache and ref count
- **Calls:** `get_resource()`, `SDL_RWFromMem()`, `SDL_RWseek()`, `SDL_ReadBE16()`, `malloc()`, `byte_swap_memory()`
- **Notes:** Searches FOND association table for font size, then loads NFNT or FONT resource; converts 1-bit bitmap to 8-bit pixmap; manages location and width tables.

### load_ttf_font
- **Signature:** `static TTF_Font *load_ttf_font(const std::string& path, uint16 style, int16 size)` (ifdef HAVE_SDL_TTF)
- **Purpose:** Load or retrieve cached TrueType font with specified style and size.
- **Inputs:** File path (or "mono" for built-in), style flags, point size
- **Outputs/Return:** TTF_Font pointer or NULL
- **Side effects:** Caches font in `ttf_font_list` with ref count; increments existing ref count
- **Calls:** `TTF_OpenFontRW()`, `TTF_OpenFont()`, `TTF_SetFontStyle()`, `TTF_SetFontHinting()`
- **Notes:** "mono" loads built-in Aleph Sans Mono Bold; applies hinting based on `smooth_text` preference.

### load_font
- **Signature:** `font_info *load_font(const TextSpec &spec)`
- **Purpose:** Main dispatcher: load TTF if spec has font names, else fall back to bitmap.
- **Inputs:** `TextSpec` with font ID and optional TrueType file paths (normal, bold, oblique, bold_oblique)
- **Outputs/Return:** Pointer to `font_info` (ttf_font_info or sdl_font_info) or NULL
- **Side effects:** Allocates `ttf_font_info` with style variants; calls `load_ttf_font()` multiple times with fallback logic
- **Calls:** `locate_font()`, `load_ttf_font()`, `load_sdl_font()`, `assert()`
- **Notes:** For TTF, tries exact file paths first; if bold/oblique missing, synthesizes with style flags; expects bold font to load at least once (assertion).

### sdl_font_info::_unload
- **Signature:** `void sdl_font_info::_unload()`
- **Purpose:** Decrement ref count and delete bitmap font if unreferenced.
- **Inputs:** None (operates on `this`)
- **Outputs/Return:** None
- **Side effects:** Modifies `font_list` cache; deallocates pixmap and struct
- **Calls:** Searches `font_list`; calls destructor via `delete`
- **Notes:** Iterates entire font_list (O(n)) to find self; calls `delete this` directly (unusual pattern).

### ttf_font_info::_unload
- **Signature:** `void ttf_font_info::_unload()` (ifdef HAVE_SDL_TTF)
- **Purpose:** Decrement ref counts for all TTF style variants and unload fonts.
- **Inputs:** None (operates on `this`)
- **Outputs/Return:** None
- **Side effects:** Modifies `ttf_font_list` cache; deallocates TTF_Font objects
- **Calls:** `TTF_CloseFont()`; calls destructor via `delete`
- **Notes:** Iterates up to 4 style slots; decrefs each variant's ref count.

### char_width (sdl_font_info and ttf_font_info overloads)
- **Signature:** `int8 sdl_font_info::char_width(uint8 c, uint16 style) const` / `int8 ttf_font_info::char_width(uint8 c, uint16 style) const`
- **Purpose:** Return glyph advance width for bitmap or TTF font.
- **Inputs:** Character code, style flags
- **Outputs/Return:** Width in pixels (int8)
- **Side effects:** None
- **Calls:** (bitmap) direct table lookup; (TTF) `TTF_GlyphMetrics()`, `mac_roman_to_unicode()`
- **Notes:** Bitmap adds 1 pixel if bold flag set; TTF queries SDL_ttf metrics.

### _text_width (multiple overloads)
- **Signature:** `uint16 sdl_font_info::_text_width(const char *text, uint16 style, bool)` / `uint16 sdl_font_info::_text_width(const char *text, size_t length, uint16 style, bool)` / TTF variants
- **Purpose:** Calculate total width of a string in bitmap or TTF font.
- **Inputs:** Text pointer (or pointer + length), style flags, utf8 flag (TTF only)
- **Outputs/Return:** Width in pixels (uint16)
- **Side effects:** None
- **Calls:** (bitmap) `char_width()` in loop; (TTF) `process_printable()` or `process_macroman()`, then `TTF_SizeUTF8()` or `TTF_SizeUNICODE()`
- **Notes:** TTF variant handles UTF-8 or Mac Roman encoding.

### _trunc_text
- **Signature:** `int sdl_font_info::_trunc_text(const char *text, int max_width, uint16 style) const` / TTF variant
- **Purpose:** Return count of characters that fit within max_width.
- **Inputs:** Text pointer, max pixel width, style flags
- **Outputs/Return:** Character count (int)
- **Side effects:** None
- **Calls:** (bitmap) `char_width()` in loop; (TTF) `mac_roman_to_unicode()`, `TTF_SizeUNICODE()` in decrement loop
- **Notes:** Bitmap stops early on overflow; TTF decrement loop from end (inefficient for long strings).

### draw_styled_text
- **Signature:** `int font_info::draw_styled_text(SDL_Surface *s, const std::string& text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- **Purpose:** Draw text with inline style codes (|b, |i, |p) and optional shadow.
- **Inputs:** SDL surface, text string and length, x/y position, pixel color, style flags, utf8 flag
- **Outputs/Return:** Total width drawn (int)
- **Side effects:** Modifies SDL surface pixels
- **Calls:** Tokenizes with `style_separator`; calls `_draw_text()` (defined elsewhere)
- **Notes:** Maintains running x offset; applies shadow by drawing black copy at +1,+1 offset.

### styled_text_width
- **Signature:** `int font_info::styled_text_width(const std::string& text, size_t length, uint16 style, bool utf8) const`
- **Purpose:** Calculate width of text with style codes (excluding shadow offset initially, then adding 1).
- **Inputs:** Text string and length, style flags, utf8 flag
- **Outputs/Return:** Width in pixels (int)
- **Side effects:** None
- **Calls:** Tokenizes with `style_separator`; calls `_text_width()` on non-style tokens; calls `update_style()`
- **Notes:** Shadow adds 1 pixel at the end (returns width+1 if shadow flag set).

### trunc_styled_text
- **Signature:** `int font_info::trunc_styled_text(const std::string& text, int max_width, uint16 style) const`
- **Purpose:** Return character count (including style codes) that fit within max_width.
- **Inputs:** Text string, max pixel width, style flags
- **Outputs/Return:** Character count (int)
- **Side effects:** None
- **Calls:** Tokenizes; calls `_trunc_text()` and `_text_width()` on text tokens
- **Notes:** Style codes counted as 2 characters each; shadow reduces max_width by 1.

### style_at
- **Signature:** `std::string font_info::style_at(const std::string& text, std::string::const_iterator pos, uint16 style) const`
- **Purpose:** Determine the effective style (bold/italic) at a given position in styled text.
- **Inputs:** Text string, iterator position within string, initial style
- **Outputs/Return:** Style string ("|b", "|i", or empty)
- **Side effects:** None
- **Calls:** Tokenizes up to `pos`; calls `update_style()`
- **Notes:** Returns only bold or italic state at that position.

### draw_text
- **Signature:** `int font_info::draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- **Purpose:** Draw plain text with optional shadow; wrapper for `_draw_text()`.
- **Inputs:** SDL surface, text pointer and length, position, color, style, utf8 flag
- **Outputs/Return:** Width drawn (int)
- **Side effects:** Modifies SDL surface pixels
- **Calls:** `_draw_text()` (defined in screen_drawing.cpp)
- **Notes:** Shadow handled here (draws black shadow at +1,+1 first).

## Control Flow Notes
**Initialization:** `initialize_fonts()` runs at engine startup, opening resource files or falling back to built-in TTF.

**Load/Cache:** `load_font()` checks cache and creates new entries as needed. Bitmap fonts ref-count by (id, size) pair. TTF fonts ref-count by (path, style, size) tuple.

**Text Rendering:** Style codes (|b, |i, |p) are parsed on-demand during width/draw/truncate calls, not stored in the font object. Shadow effects are applied by drawing text twice (dark at offset, then light at normal position).

**Unload:** `unload_font()` delegates to virtual `_unload()` on base class, which decr refs and deletes when count reaches 0.

## External Dependencies
- **Resources:** `resource_manager.h`, `byte_swapping.h` (Mac resource format, big-endian)
- **File I/O:** `FileHandler.h` (FileSpecifier), `Logging.h`
- **SDL:** `<SDL_endian.h>`, optionally SDL_ttf (TTF_Font, TTF_*) 
- **STL:** `<vector>`, `<map>`, `<string>`
- **Boost:** `<boost/tokenizer.hpp>` (text parsing)
- **External symbols:** `data_search_path`, `fix_missing_overhead_map_fonts()`, `fix_missing_interface_fonts()`, `mac_roman_to_unicode()`, `environment_preferences`, `_draw_text()` (in screen_drawing.cpp)
