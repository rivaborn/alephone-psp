# Source_Files/RenderOther/FontHandler.h

## File Purpose
Defines the `FontSpecifier` class for managing text font specifications, metrics, and rendering in the Aleph One game engine. Handles font parameter specification via XML, platform-specific font operations (macOS/SDL), and OpenGL text rendering with support for text alignment and wrapping.

## Core Responsibilities
- Manage font parameters (name set, size, style, line height adjustments)
- Compute and cache font metrics (ascent, descent, leading, character widths)
- Load and update fonts via `Update()` and XML parser integration
- Render text to screen coordinates with OpenGL (with modelview matrix manipulation)
- Draw formatted text with alignment and line wrapping (OpenGL)
- Maintain platform-specific font resources (macOS ID, SDL font_info, or OpenGL textures)
- Provide character and text width queries for layout and centering

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FontSpecifier` | class | Encapsulates font parameters, metrics, and rendering operations for a single font specification |

## Global / File-Static State
None.

## Key Functions / Methods

### Init
- **Signature:** `void Init()`
- **Purpose:** Initialize font object before any other operations; compensates for lack of proper constructor.
- **Inputs:** None (uses object's member variables)
- **Outputs/Return:** None
- **Side effects:** Calls `Update()` internally; sets up platform-specific font data
- **Calls:** `Update()`
- **Notes:** Must be called before any other method.

### Update
- **Signature:** `void Update()`
- **Purpose:** Compute derived font metrics (height, line spacing, ascent, descent, character widths) from parameters; called by `Init()` and XML parser on parameter changes.
- **Inputs:** Parameters: `NameSet`, `Size`, `Style`, `AdjustLineHeight`
- **Outputs/Return:** None (updates member fields: `Height`, `LineSpacing`, `Ascent`, `Descent`, `Leading`, `Widths[256]`)
- **Side effects:** Loads/changes platform-specific font resource (ID on macOS, `Info` pointer on SDL)
- **Calls:** Not inferable from header
- **Notes:** Must be called after changing parameters and before rendering.

### TextWidth
- **Signature:** `int TextWidth(const char *Text)`
- **Purpose:** Compute total width of a C-style string for layout/centering (e.g., map titles).
- **Inputs:** `Text` - null-terminated C string
- **Outputs/Return:** Pixel width as int
- **Side effects:** None
- **Calls:** Uses `Widths[]` table (computed by `Update()`)
- **Notes:** Sums character widths from precomputed table.

### CharWidth
- **Signature:** `int CharWidth(char c) const`
- **Purpose:** Query width of a single character.
- **Inputs:** `c` - character (cast to int for table lookup)
- **Outputs/Return:** Pixel width as int
- **Side effects:** None
- **Calls:** None (inline lookup into `Widths[256]`)

### Use
- **Signature:** `void Use()` (macOS only)
- **Purpose:** Activate this font in the active GrafPort (Quickdraw draw context); mimics macOS global font paradigm.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies global Quickdraw font state
- **Calls:** Not inferable
- **Notes:** macOS-specific; no-op on other platforms.

### OGL_Reset
- **Signature:** `void OGL_Reset(bool IsStarting)` (OpenGL builds only)
- **Purpose:** Reset OpenGL font texture and display list; prevents texture/display-list memory leaks when starting or ending an OpenGL session.
- **Inputs:** `IsStarting` - true if starting an OpenGL session, false if ending
- **Outputs/Return:** None
- **Side effects:** Deallocates/reallocates `OGL_Texture`, `TxtrID`, `DispList` members
- **Calls:** Not inferable
- **Notes:** Critical for session lifecycle management.

### OGL_Render
- **Signature:** `void OGL_Render(const char *Text)` (OpenGL builds only)
- **Purpose:** Render a C-style string in OpenGL at screen coordinates, assuming left baseline at (0,0); advances modelview matrix for subsequent characters.
- **Inputs:** `Text` - null-terminated C string
- **Outputs/Return:** None
- **Side effects:** Modifies OpenGL modelview matrix; may bind textures and execute display lists
- **Calls:** Not inferable
- **Notes:** Caller should use `glPushMatrix()`/`glPopMatrix()` to preserve original matrix; assumes screen coordinates.

### OGL_DrawText
- **Signature:** `void OGL_DrawText(const char *Text, const screen_rectangle &r, short flags)` (OpenGL builds only)
- **Purpose:** Render text with alignment and line wrapping (like `_draw_screen_text()` in screen_drawing.h); modelview matrix unaffected.
- **Inputs:** `Text` - null-terminated C string; `r` - bounding rectangle; `flags` - alignment/wrapping flags
- **Outputs/Return:** None
- **Side effects:** OpenGL rendering only (no matrix alteration)
- **Calls:** Not inferable
- **Notes:** Convenience wrapper for formatted text rendering.

---

**Static utility methods (not shown in detail):**
- `FindNextName(char *NamePtr)` ΓÇô Parse next font name in comma-separated set; returns pointer to next name or NULL.
- `FindNameEnd(char *NamePtr)` ΓÇô Find end of current font name in set.

**Operator overloads:**
- `operator==(FontSpecifier&)`, `operator!=()`, `operator=()`

---

## Control Flow Notes
- **Initialization:** `Init()` ΓåÆ `Update()` establishes font and metrics
- **XML integration:** Parsers call `Update()` after modifying parameters (e.g., `NameSet`, `Size`, `Style`)
- **Rendering:** `OGL_Render()` or `OGL_DrawText()` (OpenGL), or platform-specific rendering (macOS/SDL) via calls to `Use()` or methods in `font_info`
- **Platform lifecycle:** `OGL_Reset(true)` on session start, `OGL_Reset(false)` on session end (to prevent resource leaks)

## External Dependencies
- **cseries.h** ΓÇô Core type definitions (`uint8`, `uint32`, `int16`, etc.)
- **XML_ElementParser.h** ΓÇô Base class for XML configuration parsing
- **sdl_fonts.h** ΓÇô SDL font abstraction (`font_info` class)
- **OpenGL headers** ΓÇô Platform-specific conditional includes (`GL/gl.h` on Linux, `OpenGL/gl.h` on macOS, etc.)
- **Defined elsewhere:** `screen_rectangle` (from screen_drawing.h or similar)
