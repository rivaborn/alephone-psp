# Source_Files/RenderMain/OGL_Render.cpp

## File Purpose
OpenGL renderer implementation for the Marathon/Aleph One game engine. Manages OpenGL context lifecycle, coordinate transformations, matrix setup, texture/material blending, and 2D/3D rendering. Bridges OpenGL API with engine-level rendering requests for walls, objects, text, and UI elements.

## Core Responsibilities
- OpenGL context initialization, management, and teardown
- Coordinate system transformations (Marathon world ΓåÆ OpenGL eye space)
- Projection matrix selection and management (3D depth vs. flat screen)
- Texture blending and alpha-test configuration for materials
- Static effect (TV noise) rendering via stipple patterns or stencil buffer
- Shader callbacks for standard, glowing, and special-effect textures
- Per-vertex lighting calculations for 3D models
- Crosshair and text rendering in screen space
- 2D graphics buffer management (Mac platform)
- Platform-specific window attachment and context management

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SurfaceCoords` | struct | Stores texture coordinate vectors (U, V) and complement vectors for ray-surface intersection |
| `ShaderDataStruct` | struct | Model, skin, color, and collection data passed to shader callbacks |
| `LightingDataStruct` | struct | Directional lighting parameters (type, normal-based color matrix, opacity) for per-vertex lighting |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsActive` | bool | static | Whether OpenGL rendering context is active |
| `JustInited` | bool | static | Flag for context just created; some state may need updating |
| `SavedScreenBounds` | Rect | static | Full screen boundary rectangle |
| `SavedViewBounds` | Rect | static | Render view area rectangle |
| `ViewWidth`, `ViewHeight` | short | static | Current view dimensions in pixels |
| `CenteredWorld_2_MaraEye`, `World_2_MaraEye`, etc. | GLdouble[16] | static | Coordinate transformation matrices (4 matrices total) |
| `MaraEye_2_OGLEye` | const GLdouble[16] | static | Marathon eye ΓåÆ OpenGL eye (constant transpose of standard arrangement) |
| `Screen_2_Clip`, `OGLEye_2_Clip`, `OGLEye_2_Screen` | GLdouble[16] | static | Projection and screen conversion matrices |
| `ProjectionType` | int | static | Currently-loaded projection matrix type (enum: NONE, OpenGL_Eye, Screen) |
| `Z_Near`, `Z_Far` | const GLdouble | static | Near/far clipping plane distances |
| `Z_Buffering` | bool | static | Depth buffering enabled flag |
| `XScale`, `YScale`, `XScaleRecip`, `YScaleRecip`, `XOffset`, `YOffset` | GLdouble | static | Screen Γåö world conversion scale factors and offsets |
| `Yaw` | double | static | Player yaw angle (full circle = 1) |
| `LandscapeRescale` | double | static | Landscape scaling to match software rendering |
| `SelfLuminosity` | _fixed | static | Miner's light effect and weapon flare brightness |
| `CurrFog` | OGL_FogData* | static | Pointer to active fog parameters |
| `CurrFogColor` | GLfloat[4] | static | Current fog color (may differ in infravision mode) |
| `StaticPatterns` | GLuint[4][32] | static | Pre-generated stipple patterns for static effect; 4 color channels |
| `UseFlatStatic` | bool | static | Use flat-color vs. stippled static effect |
| `FlatStaticColor` | uint16[4] | static | RGBA color for flat static mode |
| `StaticRandom` | GM_Random | static | Random number generator for static effect pixels |
| `BlendType` | short | static | Currently-set blend mode; cached to avoid redundant GL calls |
| `RenderContext` | AGLContext | static (mac) | macOS AGL rendering context |
| `ModelRenderObject` | ModelRenderer | static | 3D model renderer instance |
| `ViewDir` | GLfloat[2] | static | Model renderer base view direction (yaw, pitch) |
| `StandardShaders` | ModelRenderShader[2] | static | Shader list: normal texture, glowing texture |
| `StaticModeShaders` | ModelRenderShader[4] | static | Multi-pass static effect shaders (0ΓÇô3) |
| `SequenceNumbers` | int[4] | static | Pass indices for static mode shader callbacks |
| `ShaderData` | ShaderDataStruct | static | Reusable struct passed to shader callbacks |
| `LightingData` | LightingDataStruct | static | Reusable lighting parameters for lighting callback |
| `Buffer2D` | vector<uint32> | static (mac) | 2D graphics strip buffer for raster operations |
| `Buffer2D_Width`, `Buffer2D_Height` | int | static | Dimensions of 2D strip buffer |

## Key Functions / Methods

### OGL_IsActive
- **Signature:** `bool OGL_IsActive()`
- **Purpose:** Return whether OpenGL is active and present
- **Inputs:** None
- **Outputs/Return:** bool ΓÇô true if both present and active; false if not present or inactive
- **Side effects:** None
- **Calls:** `OGL_IsPresent()` (defined elsewhere)
- **Notes:** Guard function for all rendering calls

### OGL_ClearScreen
- **Signature:** `bool OGL_ClearScreen()`
- **Purpose:** Clear screen to black; reset color and depth buffers
- **Inputs:** None
- **Outputs/Return:** bool ΓÇô true if active and cleared; false if inactive
- **Side effects:** Clears GL_COLOR_BUFFER_BIT and GL_DEPTH_BUFFER_BIT; on Mac, may disable AGL_SWAP_RECT
- **Calls:** `OGL_IsActive()`, `aglIsEnabled()`, `aglDisable()`, `glClearColor()`, `glClear()`
- **Notes:** On Mac, disables AGL_SWAP_RECT swap-rect optimization to ensure full screen paint

### OGL_StartRun
- **Signature:** `bool OGL_StartRun()` (non-Mac) or `bool OGL_StartRun(CGrafPtr WindowPtr)` (Mac)
- **Purpose:** Initialize OpenGL rendering context, window attachment, and all GL state
- **Inputs:** `WindowPtr` (Mac only) ΓÇô QuickDraw window port
- **Outputs/Return:** bool ΓÇô true on success; false if context creation or setup failed
- **Side effects:** Creates rendering context; loads textures; initializes shaders, fonts, vertex arrays; enables culling, alpha test, Z-buffer (if configured)
- **Calls:** `OGL_IsPresent()`, `OGL_StopRun()`, `Get_OGL_ConfigureData()`, `aglChoosePixelFormat()`, `aglCreateContext()`, `aglSetDrawable()` / `aglSetFullScreen()`, `aglSetCurrentContext()`, `aglEnable()`, `aglSetInteger()`, `setup_gl_extensions()`, `load_replacement_collections()`, `OGL_StartTextures()`, `GetOnScreenFont().OGL_Reset()`, `OGL_ResetMapFonts()`, `OGL_ResetHUDFonts()`, `OGL_ResetModelSkins()`, `PreloadTextures()`, `SetupShaders()`
- **Notes:** Long initialization; sets up Z-buffer, face culling, alpha func (GL_GREATER 0.5), blending (crossfade), client-state arrays; on Mac supports experimental fullscreen mode via aglSetFullScreen

### OGL_StopRun
- **Signature:** `bool OGL_StopRun()`
- **Purpose:** Shut down OpenGL context and mark rendering inactive
- **Inputs:** None
- **Outputs/Return:** bool ΓÇô true if was active; false if already inactive
- **Side effects:** Destroys rendering context; sets `_OGL_IsActive = false`
- **Calls:** `OGL_IsActive()`, `aglSetCurrentContext(NULL)`, `aglDestroyContext()` (Mac)
- **Notes:** Cleans up textures and context before destruction

### SetProjectionType
- **Signature:** `static void SetProjectionType(int NewProjectionType)`
- **Purpose:** Switch active projection matrix (depth-aware 3D vs. flat screen)
- **Inputs:** `NewProjectionType` ΓÇô enum (Projection_OpenGL_Eye or Projection_Screen)
- **Outputs/Return:** void
- **Side effects:** Loads new projection matrix into GL_PROJECTION; updates `ProjectionType` static
- **Calls:** `glMatrixMode()`, `glLoadMatrixd()`
- **Notes:** Short-circuits if already on requested projection type

### DoLightingAndBlending
- **Signature:** `static bool DoLightingAndBlending(rectangle_definition& RenderRectangle, bool& IsBlended, GLfloat *Color, bool& ExternallyLit)`
- **Purpose:** Configure OpenGL blending, alpha test, and color for a polygon; determine if texture is glowmappable
- **Inputs:** Reference to render rectangle; references to `IsBlended`, `Color`, `ExternallyLit` (output args)
- **Outputs/Return:** bool ΓÇô true if glowmappable; false if infravision/invisible/static
- **Side effects:** Sets GL blend mode, alpha test, color; modifies output args
- **Calls:** `TEST_FLAG()`, `glDisable()`, `glEnable()`, `glBlendFunc()`, `glAlphaFunc()`, `glColor4fv()`, etc.
- **Notes:** Handles transparency, infravision tinting, self-luminosity, externally-lit determination

### SetupStaticMode / TeardownStaticMode
- **Signature:** `static void SetupStaticMode(rectangle_definition& RenderRectangle)` / `static void TeardownStaticMode()`
- **Purpose:** Initialize / finalize static (TV-noise) effect rendering
- **Inputs:** `RenderRectangle.transfer_data` ΓÇô opacity level (0ΓÇô65535)
- **Outputs/Return:** void
- **Side effects:** Generates random stipple patterns (4 channels) or sets up stencil buffer; enables GL_POLYGON_STIPPLE or GL_STENCIL_TEST
- **Calls:** `StaticRandom.KISS()`, `StaticRandom.LFIB4()`, `glDisable()`, `glEnable()`, `glPolygonStipple()`, `glStencilFunc()`, `glStencilOp()`, etc.
- **Notes:** Two modes: flat-color or multi-pass stippled; opacity controlled by alpha channel randomness

### NormalShader
- **Signature:** `void NormalShader(void *Data)`
- **Purpose:** Shader callback for standard textured rendering
- **Inputs:** `Data` ΓÇô `ShaderDataStruct*` (model, skin, color, collection)
- **Outputs/Return:** void
- **Side effects:** Sets color; loads model skin texture; sets blend mode
- **Calls:** `glColor4fv()`, `ShaderData.ModelPtr->Use()`, `LoadModelSkin()`, `SetBlend()`
- **Notes:** Called per-material by model renderer

### GlowingShader
- **Signature:** `void GlowingShader(void *Data)`
- **Purpose:** Shader callback for glowmap (self-luminous) rendering
- **Inputs:** `Data` ΓÇô `ShaderDataStruct*`
- **Outputs/Return:** void
- **Side effects:** Sets white color with alpha; enables blending; loads glow texture
- **Calls:** `glColor4f()`, `glEnable()`, `glDisable()`, `LoadModelSkin()`, `SetBlend()`
- **Notes:** Makes texture fully bright; blended additive or via premultiplied mode

### StaticModeShader
- **Signature:** `void StaticModeShader(void *Data)`
- **Purpose:** Multi-pass shader callback for static effect rendering
- **Inputs:** `Data` ΓÇô `int*` (sequence number 0ΓÇô3)
- **Outputs/Return:** void
- **Side effects:** Calls `StaticModeIndivSetup()` to configure stencil/blend per pass; loads silhouette or generates procedural static texture
- **Calls:** `StaticModeIndivSetup()`, `LoadModelSkin()`, `glBindTexture()`, `glTexImage2D()`, `glTexEnvi()`, `glTexParameteri()`
- **Notes:** For stipple mode, all 3 passes use silhouette; for stencil, pass 2 generates noise texture

### LightingCallback
- **Signature:** `static void LightingCallback(void *Data, size_t NumVerts, GLfloat *Normals, GLfloat *Positions, GLfloat *Colors)`
- **Purpose:** Per-vertex lighting calculation callback invoked by model renderer
- **Inputs:** `Data` ΓÇô `LightingDataStruct*`; `NumVerts`, `Normals`, `Positions` ΓÇô vertex geometry; `Colors` ΓÇô output array
- **Outputs/Return:** void; fills `Colors` array with RGB (and optionally A)
- **Side effects:** Modifies `Colors` output
- **Calls:** `FindShadingColor()` (defined elsewhere)
- **Notes:** Two modes: fast (dot product with normal matrix) and per-vertex directional (fades based on normal alignment to light direction)

### OGL_RenderCrosshairs
- **Signature:** `bool OGL_RenderCrosshairs()`
- **Purpose:** Render crosshair reticle in screen space
- **Inputs:** None (reads from `GetCrosshairData()`)
- **Outputs/Return:** bool ΓÇô true if rendered; false if inactive
- **Side effects:** Draws lines/octagon in screen-space using screen projection
- **Calls:** `SetProjectionType()`, `glDisable()`, `glEnable()`, `glColor4fv()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslated()`, `glLineWidth()`, `glBegin()`, `glVertex2f()`, `glEnd()`, `glPopMatrix()`
- **Notes:** Supports RealCrosshairs (4 lines in + pattern) and Circle (octagon); translates to screen center; renders in foreground (Z near 1)

### OGL_RenderText
- **Signature:** `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r, unsigned char g, unsigned char b)`
- **Purpose:** Render text string with drop shadow in screen space
- **Inputs:** Position (BaseX, BaseY); text string; RGB color (0ΓÇô255)
- **Outputs/Return:** bool ΓÇô true if rendered; false if inactive
- **Side effects:** Creates/deletes display list; draws text with 1-pixel offset shadow
- **Calls:** `OGL_IsActive()`, `SetProjectionType()`, `glGenLists()`, `glNewList()`, `glEndList()`, `glDeleteLists()`, `glMatrixMode()`, `glPushMatrix()`, `glLoadIdentity()`, `glTranslatef()`, `glCallList()`, `glPopMatrix()`, `GetOnScreenFont().OGL_Render()`
- **Notes:** Shadow is black at offset (+1, +1); text is at offset (0, 0) in user color

### SetBlend
- **Signature:** `static void SetBlend(short _BlendType)`
- **Purpose:** Set OpenGL blend function mode with caching to avoid redundant calls
- **Inputs:** `_BlendType` ΓÇô enum (Crossfade, Add, Crossfade_Premult, Add_Premult)
- **Outputs/Return:** void
- **Side effects:** Sets GL_BLEND blend function via `glBlendFunc()`; caches `BlendType` static
- **Calls:** `glBlendFunc()`
- **Notes:** Short-circuits if already on requested blend type; reduces redundant GL calls

### OGL_Copy2D (Mac only)
- **Signature:** `bool OGL_Copy2D(GWorldPtr BufferPtr, Rect& SourceBounds, Rect& DestBounds, bool UseBackBuffer, bool FrameEnd)`
- **Purpose:** Copy 2D graphics (status bar, overhead map, terminal) from QuickDraw GWorld to OpenGL framebuffer
- **Inputs:** Source graphics world; source/destination rectangles; back/front buffer selection; frame-end swap flag
- **Outputs/Return:** bool ΓÇô true if rendered; false if inactive
- **Side effects:** Converts pixel format (16/32-bit) to RGBA; reverses row order; allocates/resizes strip buffer; performs raster blit
- **Calls:** `OGL_Get2D()`, `glDrawBuffer()`, `GetPortBounds()`, `GetGWorldPixMap()`, `LockPixels()`, `GetPixBaseAddr()`, `FillBuffer2()` / `FillBuffer4()`, `glRasterPos2s()`, `glDrawPixels()`, `aglSwapBuffers()`, `UnlockPixels()`
- **Notes:** Buffers data in 16-row strips for efficiency; flips vertically (OpenGL bottom-to-top); converts ARGB 5551/8888 to RGBA 8888 with full opacity

## Control Flow Notes
**Initialization (OGL_StartRun):**
1. Validate OpenGL presence; stop prior run if active
2. Create AGL pixel format (Mac) or setup GL extensions (Win32)
3. Create rendering context; attach to window or fullscreen mode
4. Configure OpenGL state: Z-buffer, face culling (back-face), alpha test (GL_GREATER 0.5), blending (SRC_ALPHA, ONE_MINUS_SRC_ALPHA), vertex and texture-coord arrays
5. Load replacement texture collections
6. Initialize texture manager and fonts
7. Setup model renderer and shaders
8. Preload all wall textures to avoid lazy loading stalls
9. Mark just-initialized and return success

**Per-frame flow (inferred):**
- `OGL_ClearScreen()` clears buffers each frame
- Set projection (Projection_Screen for UI, Projection_OpenGL_Eye for 3D world)
- Render 3D world geometry and models (via model renderer with shader callbacks)
- Render 2D overlays: crosshairs, text, status bar, overhead map (screen projection)
- Swap buffers at frame end (on Mac, via `aglSwapBuffers()`)

**Rendering a textured polygon:**
1. Determine blending and color via `DoLightingAndBlending()`
2. If static effect: `SetupStaticMode()` for multi-pass; render 3ΓÇô4 times with different stencil/blend
3. Otherwise: render once with standard blending and texture
4. If glowmapped and opaque enough: render glow layer additively on top

## External Dependencies
**Includes/Imports:**
- STL: `<vector>`, `<set>`, `<algorithm>`
- Standard C: `<string.h>`, `<stdlib.h>`, `<math.h>`
- Engine: `cseries.h`, `world.h`, `shell.h`, `preferences.h`, `interface.h`, `render.h`, `map.h`, `player.h`
- Rendering: `OGL_Render.h`, `OGL_Textures.h`, `AnimatedTextures.h`, `Crosshairs.h`, `VecOps.h`, `Random.h`, `ViewControl.h`, `OGL_Faders.h`, `ModelRenderer.h`, `Logging.h`
- OpenGL: `<OpenGL/gl.h>`, `<OpenGL/glu.h>` (macOS) or `<GL/gl.h>`, `<GL/glu.h>` (other); `<AGL/agl.h>` (Mac-specific)
- Platform: `OGL_Win32.h` (Windows), `my32bqd.h` (Mac)

**Symbols defined elsewhere (not visible here):**
- `OGL_IsPresent()`, `OGL_StartProgress()`, `OGL_ProgressCallback()`, `OGL_StopProgress()` ΓÇô OpenGL lifecycle
- `Get_OGL_ConfigureData()`, `TEST_FLAG()`, `SET_FLAG()` ΓÇô Configuration and bit-field macros
- `load_replacement_collections()`, `count_replacement_collections()` ΓÇô Resource management
- `OGL_StartTextures()`, `OGL_ResetTextures()` ΓÇô Texture subsystem
- `PreloadTextures()`, `PreloadWallTexture()` ΓÇô Texture preloading
- `GetOnScreenFont()`, `OGL_ResetMapFonts()`, `OGL_ResetHUDFonts()` ΓÇô Font system
- `OGL_ResetModelSkins()`, `LoadModelSkin()` ΓÇô Model skin management
- `GetCrosshairData()` ΓÇô Crosshair parameters
- `FindShadingColor()` ΓÇô Lighting color lookup table
- `setup_gl_extensions()` ΓÇô Platform GL extension initialization
