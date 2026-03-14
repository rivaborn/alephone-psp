# Source_Files/RenderOther/screen_sdl.cpp

## File Purpose
SDL-based screen management and rendering system for the Aleph One game engine. Handles display initialization, mode switching (resolution/fullscreen/acceleration), rendering surface allocation, and frame composition (world view, HUD, terminal, overhead map). Integrates with OpenGL for accelerated rendering when available.

## Core Responsibilities
- Initialize and tear down SDL display surfaces and rendering contexts
- Manage screen mode changes (resolution, fullscreen toggling, bit depth, acceleration)
- Allocate and reallocate off-screen rendering buffers (world_pixels, HUD_Buffer, Term_Buffer)
- Route frame rendering to software or OpenGL acceleration paths
- Composite game world, HUD, terminal, and overlay elements to display
- Handle color palette management and gamma correction
- Manage OpenGL initialization, viewport setup, and context lifecycle
- Scale low-resolution buffers 2x for display when hardware acceleration is disabled

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` | struct | Display configuration (size, acceleration type, fullscreen, bit depth, gamma) |
| `SDL_Surface` | struct (SDL) | Rendering surface for CPU-based drawing and display |
| `view_data` | struct | Camera/view state (FOV, position, target polygon, effects) |
| `color_table` | struct | Palette of 256 RGB colors for 8-bit rendering |
| `bitmap_definition` | struct | Bitmap metadata for rendering (width, height, pitch, row pointers) |
| `SDL_Rect` | struct (SDL) | Rectangle for blitting and viewport operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `main_surface` | `SDL_Surface*` | static | Primary display surface (mode-dependent: SDL surface or OpenGL context) |
| `world_pixels` | `SDL_Surface*` | global | Off-screen buffer for world view rendering |
| `HUD_Buffer` | `SDL_Surface*` | global | Off-screen buffer for HUD rendering |
| `Term_Buffer` | `SDL_Surface*` | global | Off-screen buffer for terminal/computer interface (640├ù320, 32-bit) |
| `PrevFullscreen` | bool | static | Tracks previous fullscreen state to detect transitions |
| `in_game` | bool | static | True if in-game screen (variable size); false if menu (640├ù480) |
| `desktop_width`, `desktop_height` | int | static | Desktop resolution for fullscreen mode calculations |
| `PrevBufferWidth`, `PrevBufferHeight` | int | static | Tracks rendering buffer size changes |
| `PrevOffsetWidth`, `PrevOffsetHeight` | int | static | Tracks viewport offset changes |
| `gl_info_printed` | bool | static | One-time flag to print OpenGL vendor/renderer/version info |
| `OGL_MapActive` | bool | extern | True if overhead map is rendering via OpenGL |
| `OGL_HUDActive` | bool | extern | True if HUD is rendering via OpenGL |
| `OGL_TermActive` | bool | (file scope) | True if terminal is rendering via OpenGL |
| `OGL_Term_Texture` | `GLuint` | (file scope) | OpenGL texture ID for terminal rendering |

## Key Functions / Methods

### initialize_screen
- **Signature:** `void initialize_screen(struct screen_mode_data *mode, bool ShowFreqDialog)`
- **Purpose:** Initialize screen system on startup; allocate color tables, view data, and initial rendering surfaces.
- **Inputs:** `mode` (desired screen configuration), `ShowFreqDialog` (unused in this file)
- **Outputs/Return:** None; modifies global state
- **Side effects:** Allocates color_table globals, world_view structure, initializes world_pixels to NULL (lazy), calls unload_all_collections if reinitializing
- **Calls:** malloc, memset, assert, change_screen_mode
- **Notes:** Only executes full initialization once (screen_initialized flag). On reinit, clears collections and surfaces. Always sets menu mode (640├ù480, no OpenGL) initially.

### change_screen_mode (low-level: int width, int height, int depth, bool nogl)
- **Signature:** `static void change_screen_mode(int width, int height, int depth, bool nogl)`
- **Purpose:** Low-level SDL video mode setting with OpenGL context initialization.
- **Inputs:** `width`, `height` (pixel dimensions), `depth` (bit depth: 8 or 16), `nogl` (disable OpenGL)
- **Outputs/Return:** None; modifies global state
- **Side effects:** Calls SDL_SetVideoMode, frees/recreates HUD_Buffer and Term_Buffer, sets OpenGL attributes, calls ReloadViewContext on Windows/macOS, prints GL info on first OpenGL use
- **Calls:** SDL_SetVideoMode, SDL_GetError, SDL_GL_SetAttribute, SDL_SetColors, SDL_SetPalette, glGetString, glScissor, glViewport, clear_screen, ReloadViewContext, OGL_StartRun
- **Notes:** Retries with 16-bit OpenGL if 24-bit fails. Term_Buffer always created as 640├ù320 32-bit RGBA. Handles endianness for buffer format.

### change_screen_mode (high-level: screen_mode_data *mode, bool redraw)
- **Signature:** `void change_screen_mode(struct screen_mode_data *mode, bool redraw)`
- **Purpose:** High-level wrapper to apply screen_mode_data configuration.
- **Inputs:** `mode` (desired config), `redraw` (if true, apply and clear; if false, just store)
- **Outputs/Return:** None
- **Side effects:** Copies mode to global screen_mode, resets frame_count/frame_index, optionally calls low-level change_screen_mode and clear_screen
- **Calls:** change_screen_mode (low-level), clear_screen, recenter_mouse
- **Notes:** Decodes ViewSizes array by mode->size to extract OverallWidth/OverallHeight.

### enter_screen
- **Signature:** `void enter_screen(void)`
- **Purpose:** Transition from menu to in-game screen; initialize OpenGL and viewport.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets in_game=true, changes screen mode from 640├ù480 to configured size/resolution, disables overhead map/terminal, resets view effects, initializes OpenGL if available, clears modifier keys, updates OGL_HUDActive
- **Calls:** set_overhead_map_status, set_terminal_status, change_screen_mode, SDL_SetModState, OGL_StartRun
- **Notes:** Clears view->effect. On MUST_RELOAD_VIEW_CONTEXT platforms (Windows, macOS), ReloadViewContext called during change_screen_mode.

### exit_screen
- **Signature:** `void exit_screen(void)`
- **Purpose:** Transition from game to menu screen; teardown OpenGL.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets in_game=false, changes to 640├ù480 menu mode, disables OpenGL (OGL_StopRun)
- **Calls:** change_screen_mode, OGL_StopRun
- **Notes:** Forced to software rendering (nogl=true) for menu.

### render_screen
- **Signature:** `void render_screen(short ticks_elapsed)`
- **Purpose:** Main per-frame rendering routine; routes rendering to correct acceleration path, composites all screen elements.
- **Inputs:** `ticks_elapsed` (time delta since last frame in ticks)
- **Outputs/Return:** None; updates display
- **Side effects:** Updates world_view from current_player state, allocates/reallocates buffers if size changed, calls render_view to render 3D world, renders crosshairs/HUD/terminal, blits to main_surface or OpenGL, swaps GL buffers. Tracks state changes (fullscreen toggle, size change) via global Prev* variables.
- **Calls:** set_overhead_map_status, set_terminal_status, render_view, Crosshairs_Render, update_fps_display, DisplayPosition, DisplayNetMicStatus, DisplayMessages, DisplayInputLine, reallocate_world_pixels, clear_screen, draw_interface, precalculate_bitmap_row_addresses, update_screen, DrawHUD, OGL_SetWindow, OGL_RenderCrosshairs, OGL_DrawHUD, OGL_SwapBuffers, OGL_Blitter
- **Notes:** Calculates BufferRect, ViewRect, HUD_DestRect based on mode.size and ViewSizes array. Handles three display modes: terminal (640├ù320), overhead map (fill available), normal (mode-dependent). Dual-rendering path: software (update_screen) or OpenGL (OGL_Blitter for 2D, OGL_SwapBuffers for GL).

### reallocate_world_pixels
- **Signature:** `static void reallocate_world_pixels(int width, int height)`
- **Purpose:** Allocate or resize the main rendering buffer.
- **Inputs:** `width`, `height` (desired buffer dimensions)
- **Outputs/Return:** None; allocates world_pixels
- **Side effects:** Frees old world_pixels, creates new SDL_CreateRGBSurface (SDL_SWSURFACE), sets color palette if 8-bit, alerts if allocation fails (fatalError, outOfMemory)
- **Calls:** SDL_FreeSurface, SDL_CreateRGBSurface, build_sdl_color_table, SDL_SetColors, alert_user
- **Notes:** Inherits format from main_surface. On 8-bit mode, applies world_color_table palette.

### update_screen
- **Signature:** `static void update_screen(SDL_Rect &source, SDL_Rect &destination, bool hi_rez)`
- **Purpose:** Blit world_pixels to main_surface display; optionally scale 2x for low-resolution rendering.
- **Inputs:** `source` (unused), `destination` (target viewport rect), `hi_rez` (if false, scale 2x)
- **Outputs/Return:** None; updates display
- **Side effects:** Locks main_surface if needed, calls quadruple_surface (pixel-replication scaling) for low-res, unlocks, calls SDL_UpdateRects
- **Calls:** SDL_BlitSurface, SDL_LockSurface, SDL_UnlockSurface, quadruple_surface (template), SDL_UpdateRects
- **Notes:** quadruple_surface is a template function duplicating each pixel 2├ù2. Dispatches on BytesPerPixel (1, 2, or 4).

### DrawHUD
- **Signature:** `void DrawHUD(SDL_Rect &dest_rect)`
- **Purpose:** Composite HUD_Buffer to main_surface (non-OpenGL path).
- **Inputs:** `dest_rect` (destination rectangle on main_surface)
- **Outputs/Return:** None
- **Side effects:** Blits HUD_Buffer to main_surface, updates display. On PSP, rescales HUD_Buffer to 480├ù360.
- **Calls:** SDL_BlitSurface, SDL_UpdateRects, rescale_surface (PSP)
- **Notes:** Source rect hardcoded as {0, 320, 640, 160} for standard platforms (lower 160 pixels of HUD_Buffer). PSP variant extracts different region and rescales.

### clear_screen
- **Signature:** `void clear_screen(void)`
- **Purpose:** Clear display to black.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Either calls OGL_ClearScreen and SDL_GL_SwapBuffers (if OpenGL), or fills main_surface black and updates
- **Calls:** SDL_GetVideoSurface, OGL_ClearScreen, SDL_GL_SwapBuffers, SDL_FillRect, SDL_UpdateRect
- **Notes:** Used at startup and during mode transitions.

### darken_world_window
- **Signature:** `void darken_world_window(void)`
- **Purpose:** Draw a semi-transparent dithering pattern over world window (for fades).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** If OpenGL, renders semi-transparent black quad; if software, locks main_surface and draws dithering pattern
- **Calls:** (OpenGL path): glPushAttrib, glDisable, glEnable, glMatrixMode, glLoadIdentity, glOrtho, glBlendFunc, glColor4f, glBegin, glVertex2i, glEnd, glPopMatrix, glPopAttrib, SDL_GL_SwapBuffers; (software): draw_pattern_rect (template), SDL_LockSurface, SDL_UnlockSurface, SDL_UpdateRects
- **Notes:** Dithering pattern is checkerboard (pixel at (x,y) drawn if (x+y)&1==0).

### start_tunnel_vision_effect
- **Signature:** `void start_tunnel_vision_effect(bool out)`
- **Purpose:** Activate or deactivate tunnel vision (sniper mode) FOV effect.
- **Inputs:** `out` (true to activate, false to deactivate)
- **Outputs/Return:** None; modifies world_view
- **Side effects:** Sets world_view->target_field_of_view to TUNNEL_VISION_FOV, EXTRAVISION_FOV, or NORMAL_FOV
- **Calls:** NetAllowTunnelVision (checks if networked game allows it)
- **Notes:** Smooth interpolation of target FOV handled elsewhere in render loop.

**Trivial helpers** (not fully documented):
- `build_sdl_color_table`: Convert color_table to SDL_Color array format (RGB shift from 16-bit to 8-bit).
- `toggle_fullscreen`, `toggle_fill_the_screen`: Convenience wrappers to change fullscreen state and reapply mode.
- `build_direct_color_table`: Create grayscale palette for direct-color (non-indexed) modes.
- `bound_screen`, `change_interface_clut`, `change_screen_clut`, `animate_screen_clut`, `assert_world_color_table`, `update_screen_window`: Palette and interface management.
- `render_computer_interface`, `render_overhead_map`: Delegate to game-world rendering functions.
- `calculate_destination_frame`, `ReloadViewContext`, `validate_world_window`: Utility accessors.

## Control Flow Notes
**Initialization (startup):**
1. `initialize_screen()` called once; allocates color tables, world_view, sets menu mode (640├ù480, no GL).

**Level entry:**
2. `enter_screen()` called; switches to in-game mode (configured resolution/fullscreen), initializes OpenGL if enabled.

**Per-frame rendering:**
3. `render_screen(ticks_elapsed)` called every frame:
   - Updates world_view from current_player state (position, facing, effects).
   - Determines viewport layout (terminal mode 640├ù320, overhead map, or normal).
   - Detects size/fullscreen changes via Prev* globals; reallocates buffers if needed.
   - Calls render_view() to render 3D world geometry into world_pixels buffer.
   - Renders overlays (crosshairs, HUD, FPS, messages) into world_pixels.
   - Composites to display: either software path (update_screen blits) or OpenGL path (OGL_Blitter/OGL_DrawHUD).
   - Swaps GL buffers if in OpenGL mode.

**Level exit:**
4. `exit_screen()` called; switches back to menu mode (640├ù480, software), stops OpenGL.

**Mode changes (anytime):**
- User-triggered `toggle_fullscreen()` or `toggle_fill_the_screen()` ΓåÆ calls high-level `change_screen_mode()` ΓåÆ calls low-level `change_screen_mode()` ΓåÆ SDL_SetVideoMode.

## External Dependencies
**Includes (notable):**
- `SDL.h`, `SDL_opengl.h` ΓÇö SDL and OpenGL APIs
- `world.h`, `map.h`, `render.h` ΓÇö Game world, map, rendering definitions
- `shell.h`, `interface.h`, `player.h` ΓÇö Core game state (current_player, game state)
- `OGL_Blitter.h`, `OGL_Render.h` ΓÇö OpenGL rendering backend
- `ViewControl.h` ΓÇö Camera/FOV utilities
- `screen_drawing.h` ΓÇö Drawing primitives
- `Crosshairs.h`, `overhead_map.h`, `computer_interface.h` ΓÇö UI/overlay rendering

**Defined elsewhere (key symbols used):**
- `current_player`, `dynamic_world` ΓÇö Global game state (defined in world/player modules)
- `world_view` ΓÇö View/camera structure (allocated here, manipulated in render.cpp)
- `render_view()` ΓÇö Main 3D rendering function (defined in render.cpp)
- `screen_mode` ΓÇö Global screen_mode_data structure (defined in shell/preferences)
- `ViewSizes[]` ΓÇö Array of layout configs (defined in game_window.c)
- `OGL_StartRun()`, `OGL_StopRun()`, `OGL_RenderCrosshairs()`, `OGL_DrawHUD()`, `OGL_SetWindow()`, `OGL_SwapBuffers()` ΓÇö OpenGL lifecycle & rendering (defined in OGL_Render.cpp)
- `set_overhead_map_status()`, `set_terminal_status()` ΓÇö Overlay state management
- `update_fps_display()`, `DisplayPosition()`, `DisplayNetMicStatus()`, `DisplayMessages()`, `DisplayInputLine()` ΓÇö HUD text rendering
