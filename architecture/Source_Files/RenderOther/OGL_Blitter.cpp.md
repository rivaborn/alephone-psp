# Source_Files/RenderOther/OGL_Blitter.cpp

## File Purpose
Implements an OpenGL texture-based blitter that converts SDL surfaces into tiled GPU textures for efficient rendering. Manages the transformation of 2D SDL surface data into OpenGL texture tiles (256├ù256), with matrix setup and render operations.

## Core Responsibilities
- Convert SDL surfaces into OpenGL textures, tiling large images into 256├ù256 chunks
- Calculate and manage scaling factors between source surface and destination screen coordinates
- Set up orthographic projection matrices for 2D rendering
- Render tiled textures as quads with proper texture coordinates and scaling
- Manage GPU texture lifecycle (creation, upload, deletion)
- Preserve and restore OpenGL state during rendering operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| SDL_Surface | external struct | Source image data to be blitted |
| SDL_Rect | external struct | Rectangle regions for source tiles and orthographic projection |
| vector<SDL_Rect> (m_rects) | container | Stores bounding rectangles for each tile in source space |
| vector<GLuint> (m_refs) | container | Stores OpenGL texture handles (one per tile) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| OGL_Blitter::tile_size | static const int | class-static | Tile dimension (256├ù256); defines texture granularity |

## Key Functions / Methods

### Constructor: `OGL_Blitter(const SDL_Surface& s, const SDL_Rect& dst, const SDL_Rect& ortho)`
- **Signature:** `OGL_Blitter(const SDL_Surface& s, const SDL_Rect& dst, const SDL_Rect& ortho)`
- **Purpose:** Initialize the blitter, convert SDL surface to tiled OpenGL textures, and upload to GPU.
- **Inputs:** `s` (source SDL surface), `dst` (destination screen rect for scaling), `ortho` (orthographic projection bounds)
- **Outputs/Return:** None (constructor)
- **Side effects:** Allocates GPU textures via `glGenTextures()`, uploads pixel data via `glTexImage2D()`, modifies GL state (texture binding, parameters)
- **Calls:** `glGenTextures()`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `glBindTexture()`, `glTexParameteri()`, `glTexImage2D()`, `SDL_FreeSurface()`
- **Notes:** Handles endianness via `#ifdef ALEPHONE_LITTLE_ENDIAN` for correct RGBA channel ordering. Tiles surface into ceil(height/256) ├ù ceil(width/256) regions. Each tile may be smaller than 256├ù256 at boundaries.

### `SetupMatrix()`
- **Signature:** `void SetupMatrix()`
- **Purpose:** Save OpenGL state and configure orthographic projection for 2D blitting.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Pushes attribute state, disables depth/culling/fog/stencil, enables blending; pushes and reconfigures projection and modelview matrices
- **Calls:** `glPushAttrib()`, `glDisable()` (multiple), `glEnable()`, `glMatrixMode()`, `glPushMatrix()`, `glLoadIdentity()`, `gluOrtho2D()`
- **Notes:** Inverts Y-axis via `gluOrtho2D(y, y+h, y+h, y)` for screen-space coordinates. Must be paired with `RestoreMatrix()`.

### `Draw()`
- **Signature:** `void Draw()`
- **Purpose:** Render all tiled textures as textured quads with proper scaling.
- **Inputs:** None (uses internal state: m_rects, m_refs, scaling factors)
- **Outputs/Return:** None (renders to framebuffer)
- **Side effects:** Modifies OpenGL state (texture binding, color, vertex submission)
- **Calls:** `glBindTexture()`, `glColor3ub()`, `glBegin()`, `glTexCoord2f()`, `glVertex3f()`, `glEnd()` (in loop)
- **Notes:** Uses legacy immediate-mode OpenGL. Loops over all tiles, computing texture-coordinate scale (`VScale`/`UScale`) for partial tiles. Applies `x_scale`, `y_scale`, `x_offset`, `y_offset` to map tile coordinates to screen space.

### `RestoreMatrix()`
- **Signature:** `void RestoreMatrix()`
- **Purpose:** Restore OpenGL state saved by `SetupMatrix()`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Pops modelview and projection matrices, restores all attributes
- **Calls:** `glPopMatrix()`, `glMatrixMode()`, `glPopAttrib()`
- **Notes:** Must be called after `SetupMatrix()` and `Draw()` to avoid state leakage.

### Destructor: `~OGL_Blitter()`
- **Signature:** `~OGL_Blitter()`
- **Purpose:** Release GPU texture resources and internal vectors.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Deletes OpenGL textures via `glDeleteTextures()`, clears vectors
- **Calls:** `glDeleteTextures()`, vector::clear()
- **Notes:** Critical for cleanup; GPU memory leaks if not called.

## Control Flow Notes
Typical usage pattern:
1. Constructor uploads SDL surface as tiled textures.
2. Caller invokes `SetupMatrix()` to prepare GL state.
3. Caller invokes `Draw()` to render tiles.
4. Caller invokes `RestoreMatrix()` to clean up GL state.
5. Destructor called on object destruction to free textures.

Fits into a **render phase** of the frame loop; assumes an active OpenGL context already exists.

## External Dependencies
- **SDL (Simple DirectMedia Layer):** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`
- **OpenGL:** `GL/gl.h`, `GL/glu.h` (platform-conditional includes); `glGenTextures()`, `glBindTexture()`, `glTexImage2D()`, matrix/projection functions, immediate-mode rendering
- **Platform conditionals:** Handles macOS (`__APPLE__`/`__MACH__`), Windows (`__WIN32__`), and Linux endianness (`ALEPHONE_LITTLE_ENDIAN`)
- **Included header:** `OGL_Blitter.h` (class definition, static tile_size)
