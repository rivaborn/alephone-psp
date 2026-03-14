# Source_Files/RenderOther/screen_drawing.cpp

## File Purpose
Manages 2D UI and HUD rendering by drawing shapes, text, rectangles, and polygons to SDL surfaces. Provides a portable interface for screen output redirection and maintains interface resources (colors, fonts, rectangles).

## Core Responsibilities
- Initialize interface resources (fonts, colors, rectangles) from data and XML
- Redirect drawing output between screen, world, HUD, and terminal buffers
- Render shapes/sprites with SDL blitting and 8-bit surface handling
- Render text with bitmap and TrueType fonts, supporting wrapping and positioning
- Draw primitives (lines, rectangles, polygons) with Cohen-Sutherland and Sutherland-Hodgman clipping
- Manage global clipping rectangle state for all drawing operations
- Parse XML configuration for interface geometry and resources

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| screen_rectangle | struct | Portable rectangle (top, left, bottom, right) for interface layout |
| FontSpecifier | class (external) | Font metadata (name, size, style) and rendering interface |
| rgb_color | struct | Color values in 16-bit RGB format |
| sdl_font_info | struct (external) | Bitmap font glyph metrics and pixmap |
| ttf_font_info | struct (external) | TrueType font wrapper; conditional on HAVE_SDL_TTF |
| XML_RectangleParser | class | SAX-style parser for rectangle XML elements |
| span_t | struct | Scan-line fill helper (left/right spans per horizontal line) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| draw_surface | SDL_Surface* | global | Current target for all drawing operations |
| old_draw_surface | SDL_Surface* | static | Saved surface for port restoration |
| draw_clip_rect_active | bool | global | Flag: whether drawing is clipped |
| draw_clip_rect | screen_rectangle | global | Current clipping bounds |
| interface_rectangles | screen_rectangle[20] | static | Hardcoded UI element bounding boxes |
| InterfaceFonts | FontSpecifier[7] | static | Interface font specifications (Monaco, Courier) |
| InterfaceColors | rgb_color[26] | static | Hardcoded interface color palette |
| RectangleParser | XML_RectangleParser | static | Reusable XML parser instance |
| va1, va2, max_vertices | world_point2d*, int | static (draw_polygon) | Temporary vertex lists for polygon clipping |
| span | span_t* | static (draw_polygon) | Scan-line fill span buffer |

## Key Functions / Methods

### initialize_screen_drawing
- Signature: `void initialize_screen_drawing(void)`
- Purpose: Bootstrap interface resources (rectangles, colors, fonts)
- Inputs: None
- Outputs: None
- Side effects: Initializes static arrays, calls `FontSpecifier::Init()` on each interface font
- Calls: `load_interface_rectangles()`, `load_screen_interface_colors()`, `InterfaceFonts[].Init()`
- Notes: Called once at engine startup; XML parsing occurs after this via separate calls

### _set_port_to_screen_window, _set_port_to_gworld, _set_port_to_HUD, _set_port_to_term
- Signature: `void _set_port_to_{X}(void)`
- Purpose: Redirect `draw_surface` to a different SDL surface (video, world, HUD, or terminal buffer)
- Inputs: None
- Outputs: None
- Side effects: Saves current surface, sets new target; asserts `old_draw_surface == NULL` to catch nesting
- Calls: `SDL_GetVideoSurface()`
- Notes: Must be paired with `_restore_port()`; used during frame rendering to composite multiple layers

### _restore_port
- Signature: `void _restore_port(void)`
- Purpose: Restore previous drawing surface after port redirection
- Inputs: None
- Outputs: None
- Side effects: Restores `draw_surface` from `old_draw_surface`, clears saved pointer

### set_drawing_clip_rectangle
- Signature: `void set_drawing_clip_rectangle(short top, short left, short bottom, short right)`
- Purpose: Enable/disable and set rectangular clipping region for all subsequent drawing
- Inputs: Top, left, bottom, right bounds (top < 0 disables clipping)
- Outputs: None
- Side effects: Updates `draw_clip_rect_active` and `draw_clip_rect` globals
- Notes: Clipping is applied by all drawing functions before pixel writes

### _draw_screen_shape
- Signature: `void _draw_screen_shape(shape_descriptor shape_id, screen_rectangle *destination, screen_rectangle *source)`
- Purpose: Blit a shape/sprite to current surface with optional source region cropping
- Inputs: Shape descriptor, destination rect, optional source rect
- Outputs: None
- Side effects: Retrieves shape surface, blits, updates video if drawing to screen, frees shape surface
- Calls: `get_shape_surface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`, `SDL_FreeSurface()`, `SDL_DisplayFormat()`
- Notes: Converts 8-bit surfaces to display format to work around SDL palette-blit limitations; returns early if shape not found

### _draw_screen_shape_at_x_y
- Signature: `void _draw_screen_shape_at_x_y(shape_descriptor shape_id, short x, short y)`
- Purpose: Blit a shape at absolute X,Y coordinates
- Inputs: Shape descriptor, x, y position
- Outputs: None
- Side effects: Blits full shape (no cropping), updates video
- Calls: `get_shape_surface()`, `SDL_BlitSurface()`, `SDL_UpdateRects()`, `SDL_FreeSurface()`

### draw_glyph (template)
- Signature: `template <class T> inline static int draw_glyph(uint8 c, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, bool oblique)`
- Purpose: Render a single glyph to frame buffer with per-pixel clipping and stylization
- Inputs: Character code, screen position, pixel pointer, pitch, clip bounds, color, font metrics, italic flag
- Outputs: Advance width (pixels to move horizontally for next glyph)
- Side effects: Writes pixels directly to frame buffer; applies kerning, ascent, bold (double-draw), underline, italic offset
- Calls: None (direct pixel writes)
- Notes: Early-exits if glyph fully clipped; handles oblique (italic) by shifting every other scanline

### draw_text (template)
- Signature: `template <class T> inline static int draw_text(const uint8 *text, size_t length, int x, int y, T *p, int pitch, int clip_left, int clip_top, int clip_right, int clip_bottom, uint32 pixel, const sdl_font_info *font, uint16 style)`
- Purpose: Render text string to frame buffer by iterating glyphs
- Inputs: Text bytes, length, position, pixel pointer, pitch, clip bounds, color, font, style flags
- Outputs: Total width of rendered text
- Side effects: Calls `draw_glyph()` for each character, skips out-of-range characters
- Notes: Skips characters outside font's first/last character range

### sdl_font_info::_draw_text (method)
- Signature: `int sdl_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool) const`
- Purpose: Render text to SDL surface using bitmap font, dispatching to template based on pixel format
- Inputs: Target surface, text, length, position, color, style, utf8 flag
- Outputs: Text width in pixels
- Side effects: Locks surface, calls template `draw_text<T>()` for matching pixel size, unlocks, updates video if drawing to screen
- Calls: `SDL_LockSurface()`, `SDL_UnlockSurface()`, `SDL_UpdateRect()`, `::draw_text<pixel8/16/32>()`
- Notes: Dispatches at 1/2/4 bytes-per-pixel; gracefully handles missing fonts (returns 0)

### ttf_font_info::_draw_text (method)
- Signature: `int ttf_font_info::_draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- Purpose: Render text using TrueType font library (SDL_ttf)
- Inputs: Target surface, text, length, position, color, style, utf8 encoding flag
- Outputs: Rendered text width in pixels
- Side effects: Renders text to temporary surface (respecting `environment_preferences->smooth_text`), blits, updates video, frees temporary surface
- Calls: `process_printable()`, `process_macroman()`, `TTF_RenderUTF8_Blended()`, `TTF_RenderUNICODE_Blended()`, `TTF_RenderUTF8_Solid()`, `TTF_RenderUNICODE_Solid()`, `SDL_BlitSurface()`, `SDL_UpdateRect()`, `SDL_FreeSurface()`, `TTF_FontAscent()`, `TTF_FontHeight()`, `SDL_GetRGB()`, `get_ttf()`
- Notes: Conditional on `HAVE_SDL_TTF`; handles both UTF-8 and MacRoman encoding; applies clipping before blit

### _draw_screen_text
- Signature: `void _draw_screen_text(const char *text, screen_rectangle *destination, short flags, short font_id, short text_color)`
- Purpose: Draw text in a bounding rectangle with wrapping, justification, and positioning
- Inputs: Text string, destination rect, alignment flags (_center_horizontal/_center_vertical/_right_justified/_top_justified/_wrap_text), font ID, color index
- Outputs: None
- Side effects: Calls `draw_text()` on current surface, recursively calls self for wrapped text, modifies local text copy
- Calls: `text_width()`, `char_width()`, `trunc_text()`, `_get_interface_color()`, `draw_text()`, *recursive `_draw_screen_text()`*
- Notes: Wrapping disables vertical centering; handles text truncation if too wide

### draw_thin_line_noclip (template)
- Signature: `template <class T> static inline void draw_thin_line_noclip(T *p, int pitch, const world_point2d *v1, const world_point2d *v2, uint32 pixel)`
- Purpose: Render thin line using DDA (Digital Differential Analyzer) without clipping
- Inputs: Pixel pointer, pitch, line endpoints, color
- Outputs: None
- Side effects: Writes pixels directly to frame buffer
- Notes: Assumes line is entirely visible; used after Cohen-Sutherland clipping

### draw_line
- Signature: `void draw_line(SDL_Surface *s, const world_point2d *v1, const world_point2d *v2, uint32 pixel, int pen_size)`
- Purpose: Draw line or thick line with Cohen-Sutherland clipping and DDA or hexagon rendering
- Inputs: Surface, endpoints, color, pen size
- Outputs: None
- Side effects: Locks surface, writes pixels, updates video
- Calls: `cs_code()`, `draw_thin_line_noclip<T>()`, `draw_polygon()` (for thick lines)
- Notes: Pen size 1 uses DDA with clipping; thick lines (pen_size > 1) converted to hexagon and rasterized; swaps endpoints to ensure downward direction

### draw_polygon
- Signature: `void draw_polygon(SDL_Surface *s, const world_point2d *vertex_array, int vertex_count, uint32 pixel)`
- Purpose: Render filled convex polygon with Sutherland-Hodgman clipping and scan-line fill
- Inputs: Surface, vertex array, count, color
- Outputs: None
- Side effects: Dynamically allocates temporary vertex lists and span buffer, fills pixels, updates video
- Calls: Sutherland-Hodgman clipping macros (clip_min/clip_max), `SDL_FillRect()`, `SDL_UpdateRect()`
- Notes: Allocates once, reuses on subsequent calls if vertex count <= max; span buffer sized to surface height

### _fill_rect / _fill_screen_rectangle / _erase_screen
- Signature: `void _fill_rect(screen_rectangle *rectangle, short color_index)` / `void _erase_screen(short color_index)`
- Purpose: Fill rectangle or entire screen with color
- Inputs: Rectangle (or NULL for full screen), color index
- Outputs: None
- Side effects: Fills on `draw_surface`, updates video
- Calls: `_get_interface_color()`, `SDL_FillRect()`, `SDL_UpdateRects()` / `SDL_UpdateRect()`

### _frame_rect
- Signature: `void _frame_rect(screen_rectangle *rectangle, short color_index)`
- Purpose: Draw outlined rectangle (not filled) using line algorithm
- Inputs: Rectangle, color index
- Outputs: None
- Side effects: Draws four lines at edges, updates video
- Calls: `_get_interface_color()`, `draw_rectangle()`

### get_interface_rectangle / get_interface_color / get_interface_font
- Signature: `screen_rectangle *get_interface_rectangle(short index)` / `const rgb_color &get_interface_color(short index)` / `FontSpecifier &get_interface_font(short index)`
- Purpose: Accessor functions for interface resources
- Inputs: Resource index
- Outputs: Pointer/reference to resource
- Side effects: None
- Calls: None
- Notes: Include bounds-checking assertions

### GetInterfaceFont / GetInterfaceStyle / _get_font_line_height / _text_width
- Purpose: Convenience accessors for font info (used by computer_interface.cpp and others)
- Notes: Small helper functions; `_get_font_spec()` returns NULL spec stub

### _get_interface_color / _get_player_color
- Signature: `void _get_interface_color(size_t color_index, SDL_Color *color)` / `void _get_player_color(size_t color_index, SDL_Color *color)`
- Purpose: Convert internal RGB color format to SDL_Color
- Inputs: Color index (0ΓÇô25 for interface, player offset for players)
- Outputs: Pointer to SDL_Color struct to fill
- Side effects: None
- Calls: `SDL_GetRGB()` (TTF version only)
- Notes: Shifts 16-bit values right by 8 bits to get 8-bit components; player colors offset from PLAYER_COLOR_BASE_INDEX

### XML_RectangleParser::Start / HandleAttribute / AttributesDone
- Purpose: SAX parser for `<rect>` XML elements
- **Start**: Reset parsing state (IsPresent[] flags)
- **HandleAttribute**: Parse index, top, left, bottom, right attributes; validates bounds
- **AttributesDone**: Checks all five attributes present, stores rectangle into RectList at Index
- Calls: `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`
- Notes: Only used if load_interface_rectangles() and load_screen_interface_colors() are non-empty

### InterfaceRectangles_GetParser
- Signature: `XML_ElementParser *InterfaceRectangles_GetParser()`
- Purpose: Return configured parser for rectangles XML
- Returns: Pointer to static RectangleParser with RectList and NumRectangles set

### SetColorFontParserToScreenDrawing
- Signature: `void SetColorFontParserToScreenDrawing()`
- Purpose: Register screen-drawing color and font arrays with XML parsers
- Calls: `Color_SetArray()`, `Font_SetArray()`
- Notes: Called after parser setup so XML tags can populate these arrays

## Control Flow Notes
**Initialization phase** (once): `initialize_screen_drawing()` ΓåÆ load rectangles/colors/fonts ΓåÆ `SetColorFontParserToScreenDrawing()` ΓåÆ XML parsing loads customizations.

**Frame rendering phase** (per frame): Port redirection (`_set_port_to_*()`) ΓåÆ HUD composition (`_draw_screen_shape()`, `_draw_screen_text()`, drawing primitives) ΓåÆ `_restore_port()`.

**Drawing operations**: All drawing functions check `draw_clip_rect_active` and apply clipping before pixel writes or surface operations.

## External Dependencies
- **SDL**: `SDL.h`, `SDL_ttf.h` (TrueType rendering, conditional)
- **Engine core**: cseries.h, map.h, interface.h, shell.h, screen.h, fades.h
- **XML parsing**: XML_ElementParser.h, ColorParser.h, FontHandler.h
- **Font/glyph rendering**: sdl_fonts.h (sdl_font_info, font_info, TextSpec)
- **Shapes**: shape_descriptors.h, `get_shape_surface()` (defined elsewhere)

**External symbols**: `world_pixels`, `HUD_Buffer`, `Term_Buffer` (screen_sdl.cpp); `environment_preferences` (prefs); font/color style constants (styleBold, styleItalic, etc.); SDL and TTF library functions; `_get_interface_color()` overloads in shell.h.
