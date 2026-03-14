# Source_Files/RenderMain/OGL_Render.h

## File Purpose
Public interface for OpenGL 3D rendering in the Aleph One game engine (Marathon port). Declares functions to manage OpenGL contexts, rendering state, and draw game objects (walls, sprites, text, HUD). Separated from OGL_Control because this header is used by rendering code.

## Core Responsibilities
- OpenGL context lifecycle (initialization, activation checks, startup/shutdown)
- Rendering state management (viewport bounds, view parameters, buffer swapping)
- Drawing 3D geometry (walls, sprites, crosshairs)
- Drawing 2D overlays (text, HUD, status bar, overhead map via OGL_Copy2D)
- Foreground object rendering (weapons in hand)
- Special effects (infravision tinting, render modes)
- Cross-platform abstraction (Mac-specific variants handled with `#ifdef mac`)

## Key Types / Data Structures
None defined in this file. The file references opaque types defined elsewhere:

| Name | Kind | Purpose |
|------|------|---------|
| `view_data` | struct (external) | 3D view parameters (position, orientation) |
| `polygon_definition` | struct (external) | Wall geometry definition |
| `rectangle_definition` | struct (external) | Sprite/2D quad geometry |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_IsActive
- **Signature:** `bool OGL_IsActive()`
- **Purpose:** Test whether OpenGL rendering is currently active
- **Inputs:** None
- **Outputs/Return:** `true` if active, `false` otherwise; always `false` if OpenGL not present
- **Side effects:** None
- **Notes:** Idempotent query

### OGL_StartRun / OGL_StopRun
- **Signature:** `bool OGL_StartRun([CGrafPtr WindowPtr])` (Mac variant takes window pointer); `bool OGL_StopRun()`
- **Purpose:** Create/destroy OpenGL rendering context and begin/end a rendering session
- **Inputs:** `WindowPtr` (Mac only) ΓÇô graphics port for window
- **Outputs/Return:** `true` on success
- **Side effects:** Allocates/frees GPU resources; creates rendering context
- **Calls:** Likely calls platform-specific OpenGL initialization
- **Notes:** Cross-platform; Mac version requires window pointer, non-Mac version may use global/cached context

### OGL_SetWindow
- **Signature:** `bool OGL_SetWindow(Rect &ScreenBounds, Rect &ViewBounds, bool UseBackBuffer)`
- **Purpose:** Configure rendering viewport and buffer allocation
- **Inputs:** Screen bounds, view bounds (pixel rects), back buffer flag
- **Outputs/Return:** `true` on success
- **Side effects:** Allocates/reallocates back buffer if requested
- **Notes:** Allows dynamic reconfiguration of rendering bounds

### OGL_SetView
- **Signature:** `bool OGL_SetView(view_data &View)`
- **Purpose:** Set camera/perspective parameters for 3D rendering
- **Inputs:** View data (camera position, orientation, FOV, etc.)
- **Outputs/Return:** `true` on success
- **Side effects:** Updates OpenGL projection/modelview matrices
- **Notes:** Must be called before rendering 3D geometry each frame

### OGL_RenderWall / OGL_RenderSprite
- **Signature:** `bool OGL_RenderWall(polygon_definition& RenderPolygon, bool IsVertical)` and `bool OGL_RenderSprite(rectangle_definition& RenderRectangle)`
- **Purpose:** Render wall polygons and sprite quads (inhabitants, ammo, etc.)
- **Inputs:** Geometry definition; wall verticality flag
- **Outputs/Return:** `true` on success
- **Side effects:** Issues OpenGL draw calls, modifies depth buffer
- **Calls:** OpenGL state changes and draw commands (not visible here)

### OGL_Copy2D
- **Signature:** `bool OGL_Copy2D(GWorldPtr BufferPtr, Rect& SourceBounds, Rect& DestBounds, bool UseBackBuffer, bool FrameEnd)` (Mac only)
- **Purpose:** Blit 2D display elements (HUD, map, terminal) from GWorld into OpenGL framebuffer
- **Inputs:** Source GWorld pointer, source/dest rects, back buffer flag, frame-completion flag
- **Outputs/Return:** `true` on success
- **Side effects:** May complete frame presentation if `FrameEnd=true`
- **Notes:** Mac-only; handles composition of 2D and 3D rendering

### OGL_RenderText
- **Signature:** `bool OGL_RenderText(short BaseX, short BaseY, const char *Text, unsigned char r=0xff, unsigned char g=0xff, unsigned char b=0xff)`
- **Purpose:** Render a C string as text overlay
- **Inputs:** Screen position (BaseX, BaseY), text string, RGB color (defaults to white)
- **Outputs/Return:** `true` on success
- **Side effects:** Renders to framebuffer

### OGL_SwapBuffers
- **Signature:** `bool OGL_SwapBuffers()`
- **Purpose:** Swap front/back buffers to present rendered frame
- **Outputs/Return:** `true` on success
- **Side effects:** Vsyncs and displays framebuffer to screen

### OGL_SetInfravisionTint
- **Signature:** `bool OGL_SetInfravisionTint(short Collection, bool IsTinted, float Red, float Green, float Blue)`
- **Purpose:** Enable/disable and configure infravision color overlay for a shapes collection
- **Inputs:** Collection ID, tint enable flag, RGB values (0.0ΓÇô1.0)
- **Outputs/Return:** `true` on success

### OGL_SetForeground / OGL_SetForegroundView
- **Signature:** `bool OGL_SetForeground()` and `bool OGL_SetForegroundView(bool HorizReflect)`
- **Purpose:** Switch to 2D overlay mode for weapons in hand; apply horizontal mirroring
- **Inputs:** Horizontal reflection flag
- **Outputs/Return:** `true` on success
- **Notes:** Likely changes projection matrix to orthographic/HUD space

### OGL_StartMain / OGL_EndMain
- **Signature:** `bool OGL_StartMain()` and `bool OGL_EndMain()`
- **Purpose:** Scope rendering of main 3D view (begin/end main view rendering block)
- **Outputs/Return:** `true` on success

## Control Flow Notes
Typical frame flow:
1. **Init:** `OGL_StartRun()` at application startup
2. **Per-frame:**
   - `OGL_SetView(view)` ΓÇô configure camera
   - `OGL_StartMain()` ΓÇô begin 3D scene
   - `OGL_RenderWall()`, `OGL_RenderSprite()` ΓÇô draw geometry
   - `OGL_RenderCrosshairs()` ΓÇô HUD crosshair
   - `OGL_EndMain()` ΓÇô end 3D scene
   - `OGL_SetForeground()` ΓåÆ `OGL_RenderWall()`/`OGL_RenderSprite()` ΓÇô weapon rendering
   - `OGL_Copy2D()` ΓÇô composite status bar, map (Mac)
   - `OGL_RenderText()` ΓÇô debug/HUD text
   - `OGL_SwapBuffers()` ΓÇô present frame
3. **Shutdown:** `OGL_StopRun()` at exit

## External Dependencies
- **Includes:** `OGL_Setup.h` (configuration, data structures, extension checks)
- **Opaque types:** `view_data`, `polygon_definition`, `rectangle_definition` (defined elsewhere)
- **Platform types:** `CGrafPtr`, `GWorldPtr`, `Rect` (Mac Toolbox / Quickdraw; conditional on `#ifdef mac`)
- **Assumed symbols:** OpenGL context management, GPU draw calls (implementation in corresponding .cpp file)
