# Source_Files/RenderOther/FontHandler.cpp

## File Purpose
Implements cross-platform font handling for the Aleph One game engine, supporting MacOS Quickdraw, SDL, and OpenGL rendering. Manages font specifications (name, size, style), text metrics, and provides XML configuration for fonts with layout support (wrapping, alignment, truncation).

## Core Responsibilities
- Font specification and initialization (name sets, size, style, file paths)
- Platform-specific font loading: MacOS Quickdraw font ID resolution, SDL font resource loading
- Text metrics calculation: character widths, line spacing, ascent/descent caching
- OpenGL font texture generation: glyph-to-texture atlas layout, display list compilation
- OpenGL text rendering: character display lists, modelview transformations, state management
- Text layout: wrapping at word boundaries, horizontal/vertical alignment, truncation
- XML parsing and dynamic font reconfiguration via FontList updates

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FontSpecifier | struct | Font parameters (NameSet, Size, Style, File) and derived metrics (Height, LineSpacing, Ascent, Descent, Widths[256]) |
| XML_FontParser | class | XML element parser for font definitions; manages indexed FontList array updates |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| FontParser | XML_FontParser | static | Singleton parser instance shared across all font XML configuration |

## Key Functions / Methods

### FontSpecifier::Init()
- Signature: `void Init()`
- Purpose: One-time initialization; sets up platform-specific state and triggers metric loading
- Inputs: None (uses member variables)
- Outputs/Return: None
- Side effects: Clears Info (SDL), OGL_Texture (OpenGL); calls Update()
- Calls: Update()
- Notes: Must be called before any other FontSpecifier methods; called by XML parser initialization

### FontSpecifier::Update()
- Signature: `void Update()`
- Purpose: Load and cache font metrics; platform-specific implementation
- Inputs: NameSet, Size, Style, File, AdjustLineHeight
- Outputs/Return: None
- Side effects: Populates Height, LineSpacing, Ascent, Descent, Leading, Widths[256]; loads/unloads font resources
- Calls: GetFont(), GetFNum(), Use(), GetFontInfo(), CharWidth() [MacOS]; load_font(), char_width() [SDL]
- Notes: MacOS: parses comma/semicolon-delimited font name list with fallback to system font (ID=0); SDL: supports "#ID" syntax for built-in fonts or file paths

### FontSpecifier::Use()
- Signature: `void Use()` [MacOS only]
- Purpose: Set current font in the active MacOS GrafPort
- Inputs: ID, Size, Style
- Outputs/Return: None
- Side effects: Calls TextFont(), TextFace(), TextSize() on current port
- Calls: TextFont(), TextFace(), TextSize()
- Notes: Quickdraw-specific; no effect on SDL/OpenGL platforms

### FontSpecifier::TextWidth()
- Signature: `int TextWidth(const char *Text)`
- Purpose: Calculate pixel width of a C-string
- Inputs: Text (C-string, max 255 chars)
- Outputs/Return: Width in pixels
- Side effects: Saves/restores font state on MacOS
- Calls: Use(), ::TextWidth() [MacOS]; sums Widths[] array [SDL]
- Notes: MacOS version caches state; SDL version uses pre-calculated character widths from Update()

### FontSpecifier::OGL_Reset()
- Signature: `void OGL_Reset(bool IsStarting)`
- Purpose: Generate OpenGL texture atlas and display lists for all 256 characters
- Inputs: IsStarting (true=session start, false=mid-session reset), font metrics (Ascent, Descent, Widths[])
- Outputs/Return: None
- Side effects: Allocates/deallocates OGL_Texture buffer (LA88 format); calls glGenTextures(), glTexImage2D(), glGenLists(); manages GPU resources
- Calls: NextPowerOfTwo() [layout]; NewGWorld(), DrawChar(), SetGWorld() [MacOS]; SDL_CreateRGBSurface(), draw_text(), SDL_FreeSurface() [SDL]; glDeleteTextures(), glDeleteLists(), glBindTexture(), glTexParameteri(), glTexImage2D() [OpenGL]
- Notes: Pads glyphs by 1 pixel to prevent edge artifacts; bins glyphs row-by-row in power-of-two texture; IsStarting prevents double-delete of GPU resources

### FontSpecifier::OGL_Render()
- Signature: `void OGL_Render(const char *Text)`
- Purpose: Render a C-string using pre-compiled display lists
- Inputs: Text (C-string, max 255 chars)
- Outputs/Return: None
- Side effects: Binds texture; calls display lists; manages GL state (push attrib, enable texture/blend, disable cull/alpha-test, pop attrib)
- Calls: glPushAttrib(), glEnable(), glDisable(), glBindTexture(), glCallList(), glPopAttrib()
- Notes: Assumes left baseline at (0,0); modifies modelview matrix; caller should wrap with glPushMatrix()/glPopMatrix(); no-op if OGL_Texture is NULL

### FontSpecifier::OGL_DrawText()
- Signature: `void OGL_DrawText(const char *text, const screen_rectangle &r, short flags)`
- Purpose: High-level OpenGL text rendering with wrapping, alignment, truncation
- Inputs: text (C-string), r (destination rectangle), flags (_wrap_text, _center_horizontal, _center_vertical, _right_justified, _top_justified, _bottom_justified)
- Outputs/Return: None
- Side effects: Recursive for wrapped lines; modifies/restores modelview matrix; truncates or wraps text to fit rectangle
- Calls: strlen(), CharWidth(), TextWidth(), OGL_DrawText() [recursion], glMatrixMode(), glPushMatrix(), glLoadIdentity(), glTranslated(), OGL_Render(), glPopMatrix()
- Notes: Wrapping breaks at last space before width limit; vertical centering disabled when wrapping; bottom-justified by default; truncates overflow with ellipsis-style (no marker)

## Control Flow Notes
**Initialization phase:** Init() ΓåÆ Update() loads font resources and caches metrics once at engine startup or when XML config is applied.
**Configuration phase:** XML parser dynamically updates FontList; AttributesDone() calls Update() to reload affected font.
**OpenGL rendering phase:** OGL_Reset() compiles texture atlas and display lists once per font change; OGL_Render()/OGL_DrawText() reuse cached display lists every frame, only modifying modelview matrix.
**Frame-independent:** Font metrics are static per FontSpecifier; no per-frame state changes except graphics transforms.

## External Dependencies
- **OpenGL:** glGenTextures(), glTexImage2D(), glBindTexture(), glCallList(), glMatrixMode(), glTranslate(), glPush/PopMatrix(), glPush/PopAttrib(), glEnable(), glDisable()
- **MacOS Quickdraw:** TextFont(), TextFace(), TextSize(), GetFNum(), GetFontInfo(), CharWidth(), DrawChar(), GetFont(), SetFont(), GWorld/PixMap management
- **SDL:** SDL_CreateRGBSurface(), SDL_FillRect(), SDL_MapRGB(), SDL_FreeSurface(), load_font(), char_width(), draw_text() [defined in screen_drawing_sdl.cpp]
- **Game framework:** shape_descriptors.h, screen_drawing.h (screen_rectangle, RECTANGLE_WIDTH/HEIGHT macros)
- **XML parsing:** XML_ElementParser (base class), StringsEqual(), ReadBoundedInt16Value(), ReadInt16Value()
- **Utilities:** FindNextName(), FindNameEnd() (static string parsing helpers for font name lists); operator==(), operator=() (FontSpecifier comparison/assignment)
