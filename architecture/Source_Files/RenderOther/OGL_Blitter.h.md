# Source_Files/RenderOther/OGL_Blitter.h

## File Purpose
OpenGL utility class for blitting (rendering) SDL surfaces to the framebuffer. Optimizes large images by tiling them into 256├ù256 chunks and managing OpenGL texture state during rendering.

## Core Responsibilities
- Encapsulate OpenGL matrix setup/teardown for blitting operations
- Tile SDL surfaces into manageable texture sizes (256├ù256)
- Track texture references and blit rectangles for multi-tile draws
- Handle platform-specific OpenGL header inclusion (macOS, Linux, Windows)
- Coordinate SDL surface data with OpenGL rendering pipeline

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Blitter` | class | Main blit utility; manages lifecycle, matrix state, and tile rendering |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tile_size` | static const int | class-static | Tile granularity constant (256 pixels) |

## Key Functions / Methods

### Constructor
- **Signature:** `OGL_Blitter(const SDL_Surface& s, const SDL_Rect& dst, const SDL_Rect& ortho)`
- **Purpose:** Initialize blitter with source surface and rendering parameters.
- **Inputs:** SDL_Surface reference (source image), dst rect (destination viewport), ortho rect (orthographic projection bounds)
- **Outputs/Return:** None (constructor)
- **Side effects:** Allocates texture references and tile rectangles; modifies OpenGL texture state.
- **Calls:** Not visible in header.
- **Notes:** Constructor likely tiles the source surface and uploads textures.

### SetupMatrix
- **Signature:** `void SetupMatrix()`
- **Purpose:** Configure OpenGL projection and modelview matrices for blitting.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Modifies OpenGL matrix state.
- **Calls:** OpenGL matrix functions (not visible in header).
- **Notes:** Must be called before Draw(); coordinates scaling and offset parameters.

### Draw
- **Signature:** `void Draw()`
- **Purpose:** Render all tiled texture quads to the framebuffer.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Issues OpenGL draw calls; uses texture references in `m_refs` and rectangles in `m_rects`.
- **Calls:** OpenGL draw functions (not visible in header).
- **Notes:** Relies on SetupMatrix() being called first.

### RestoreMatrix
- **Signature:** `void RestoreMatrix()`
- **Purpose:** Restore OpenGL matrix state to pre-blitting conditions.
- **Inputs:** None
- **Outputs/Return:** None (void)
- **Side effects:** Modifies OpenGL matrix state.
- **Calls:** OpenGL matrix functions (not visible in header).
- **Notes:** Called after Draw() to clean up; inverse of SetupMatrix().

### Destructor
- **Signature:** `~OGL_Blitter()`
- **Purpose:** Release texture references and tile data.
- **Inputs:** None
- **Outputs/Return:** None (destructor)
- **Side effects:** Deletes OpenGL textures; clears `m_rects` and `m_refs` vectors.
- **Calls:** Not visible in header.
- **Notes:** Cleans up GPU resources.

## Control Flow Notes
Typical usage: construct ΓåÆ SetupMatrix() ΓåÆ Draw() ΓåÆ RestoreMatrix() ΓåÆ destruct. Fits into render loop as a utility for 2D image composition. Scaling and offset parameters (x_scale, y_scale, x_offset, y_offset) enable flexible viewport mapping.

## External Dependencies
- **OpenGL:** `<OpenGL/gl.h>`, `<OpenGL/glu.h>` (macOS); `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glext.h>` (Linux/Windows)
- **SDL:** `<SDL/SDL.h>` (surface and rect types)
- **Engine:** `cseries.h` (common type definitions and macros)
- **Standard Library:** `<vector>` (tile storage)
