# Source_Files/RenderOther/sdl_fonts.h

## File Purpose
Defines font rendering abstractions for the Aleph One engine, supporting both bitmap-based and TrueType fonts via SDL. Provides unified interface for text measurement, drawing, truncation, and styled text handling.

## Core Responsibilities
- Abstract interface for font operations (metrics, text rendering, width calculation)
- Bitmap font implementation using pre-loaded pixmaps and kerning tables
- TrueType font support (when SDL_TTF enabled) with multiple style variants (bold, italic, underline)
- Text layout utilities: styled text handling, width measurement, text truncation
- Font lifecycle management (load/unload with reference counting)
- UTF-8 and MacRoman text encoding support

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `font_info` | Abstract class | Base interface for all font implementations; defines metrics queries and text rendering contract |
| `sdl_font_info` | Class (extends font_info) | Bitmap font using pre-rendered pixmap; stores character metrics and location/width tables |
| `ttf_font_info` | Class (extends font_info) | TrueType font wrapper; manages multiple TTF_Font instances for different styles |
| `ttf_font_key_t` | Typedef (boost tuple) | Cache key: (font filename, size, adjust_height) for TrueType fonts |

## Global / File-Static State
None. State is encapsulated in sdl_font_info and ttf_font_info instances.

## Key Functions / Methods

### font_info (abstract interface)

#### get_ascent / get_height / get_descent / get_line_height / get_leading
- Signature: `uint16 get_ascent/height/descent() const; int16 get_leading() const;`
- Purpose: Query font metrics needed for layout
- Outputs/Return: Font metric in pixels
- Notes: Pure virtual; subclasses provide actual measurements

#### draw_text
- Signature: `int draw_text(SDL_Surface *s, const char *text, size_t length, int x, int y, uint32 pixel, uint16 style, bool utf8) const`
- Purpose: Render plain text to SDL surface
- Inputs: Target surface, text (C-string or UTF-8), position, color, style flags
- Outputs/Return: Bytes written or error code
- Side effects: Modifies SDL_Surface pixels
- Calls: `_draw_text()` (virtual implementation)

#### text_width
- Signature: `uint16 text_width(const char *text, size_t length, uint16 style, bool utf8) const`
- Purpose: Measure rendered text width in pixels
- Inputs: Text string, style flags
- Outputs/Return: Width in pixels
- Calls: `_text_width()` (virtual implementation)

#### trunc_text
- Signature: `int trunc_text(const char *text, int max_width, uint16 style) const`
- Purpose: Find truncation point for text fitting within pixel width
- Inputs: Text, max pixel width, style
- Outputs/Return: Byte offset to truncation point
- Calls: `_trunc_text()` (virtual implementation)

#### char_width
- Signature: `int8 char_width(uint8 c, uint16 style) const`
- Purpose: Query width of single character with style
- Notes: Pure virtual; enables kerning/proportional spacing

#### draw_styled_text / styled_text_width / trunc_styled_text
- Purpose: Styled text variants supporting inline style codes (bold, italic, etc.) within text
- Inputs: std::string with embedded style markup, initial style
- Outputs/Return: Rendered bytes / pixel width / truncation offset
- Notes: Parses style transitions within text string

### sdl_font_info (bitmap implementation)

#### Constructor / Destructor
- Initializes member tables; destructor frees pixmap if allocated
- Side effects: Releases allocated pixmap memory
- Notes: `ref_count` and `LoadedResource` handle resource ownership

#### _draw_text / _text_width / _trunc_text
- Purpose: Concrete bitmap rendering using pre-loaded pixmap, location_table, and width_table
- Notes: Character lookup via `location_table[char]`; width via `width_table[char]`

#### char_width
- Signature: `int8 char_width(uint8 c, uint16 style) const`
- Purpose: Lookup character width (with optional kerning adjustment)

### ttf_font_info (TrueType implementation, HAVE_SDL_TTF only)

#### Constructor
- Initializes `m_styles[styleUnderline]` array to null
- Notes: Array holds separate TTF_Font handles for each style variant

#### get_ascent / get_height / get_line_height / get_descent / get_leading
- Signature: `uint16 get_ascent() const { return TTF_FontAscent(m_styles[styleNormal]); }`
- Purpose: Delegate to SDL_ttf metrics functions on normal (non-bold, non-italic) font
- Outputs/Return: Metric from TTF_Font
- Notes: `get_line_height()` adds `m_adjust_height` offset

#### _draw_text / _text_width / _trunc_text
- Purpose: Concrete TrueType rendering via SDL_ttf functions
- Calls: `process_printable()`, `process_macroman()`, `TTF_FontAscent()`, etc.

#### get_ttf (private helper)
- Signature: `TTF_Font *get_ttf(uint16 style) const`
- Purpose: Select style variant (bold | italic) from `m_styles[]` array
- Notes: Masks style to only bold/italic bits

## Control Flow Notes
Font loading/unloading is managed externally via `initialize_fonts()`, `load_font()` (returns pointer), and `unload_font()` (accepts pointer). No visible frame/render loop dependency in this headerΓÇöfonts are typically loaded at startup and queried during text rendering elsewhere in the engine.

## External Dependencies
- **FileHandler.h**: `LoadedResource` class (manages resource lifetime for bitmap font pixmap)
- **SDL_ttf.h**: Conditional, provides `TTF_Font` type and metrics/rendering functions (when `HAVE_SDL_TTF` defined)
- **boost/tuple/tuple.hpp**: Conditional, defines `ttf_font_key_t` for font caching
- **Standard library**: `<string>`, `<cstddef>` (implicit from includes)
- **tags.h** (via FileHandler.h): Typecode definitions
