# Source_Files/RenderOther/HUDRenderer_OGL.cpp

## File Purpose
Implements OpenGL-based rendering of the game HUD (Heads-Up Display) for the Aleph One engine. Manages loading the static HUD backdrop texture from resources and rendering both static background and dynamic HUD elements (weapons display, ammo, shield, oxygen, motion sensor, text).

## Core Responsibilities
- Load and cache the static HUD backdrop texture on demand, tiled into 6 optimally-sized GL textures for efficient rendering
- Render the HUD background each frame and coordinate rendering of all dynamic overlay elements
- Provide drawing primitives for HUD elements: bitmapped shapes, textures, colored text, filled/framed rectangles
- Manage the motion sensor display with circular clip-plane approximation for rendering blips
- Handle platform-specific pixel format conversions (1555 ARGB on Mac, 8888 ARGB, SDL surfaces)
- Reset font and texture caches during initialization or when resources need reloading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| HUD_OGL_Class | class | Inherits from HUD_Class; provides OpenGL-specific methods to draw shapes, textures, text, and primitives |
| TextureManager | class | Sets up texture state, handles UV coordinate transforms, and manages rendering for individual texture draws (defined elsewhere) |
| LoadedResource | struct | Resource handle wrapper for loaded picture data; platform-specific implementation (defined in images.h context) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| HUD_OGL | HUD_OGL_Class | static | Singleton instance of the HUD renderer |
| txtr_id | GLuint[6] | static | OpenGL texture IDs for the 6 HUD backdrop tiles (layout: 256├ù128, 256├ù128, 128├ù128 top row; 256├ù32, 256├ù32, 128├ù32 bottom) |
| hud_pict_loaded | bool | static | Whether the HUD backdrop texture is loaded and cached in VRAM |
| hud_pict_not_found | bool | static | Flag to skip repeated load attempts if the HUD picture resource is missing |
| MotionSensorActive | bool | extern | Global flag indicating whether the motion sensor should update and render |

## Key Functions / Methods

### OGL_DrawHUD
- Signature: `void OGL_DrawHUD(Rect &dest, short time_elapsed)`
- Purpose: Main entry point; loads HUD texture on first call, then renders background and coordinates dynamic element updates each frame
- Inputs: `dest` (screen rectangle for HUD placement), `time_elapsed` (time delta, or NONE to reset/unload textures)
- Outputs/Return: void; renders to GL framebuffer
- Side effects: Loads/caches HUD texture, marks all dynamic displays dirty, sets GL scissor region and projection matrix, modifies GL state (enables textures, disables depth/blend/fog)
- Calls: `get_picture_resource_from_images()`, `picture_to_surface()`, `glGenTextures()`, `glTexImage2D()`, `mark_*_as_dirty()`, `HUD_OGL.update_everything()`
- Notes: Converts platform pixel formats (Mac 1555/8888 ARGB or SDL) to 8888 RGBA for GL. Unloads when `time_elapsed == NONE`. Conditionally disables textures if Lua texture palette is active.

### OGL_ResetHUDFonts
- Signature: `void OGL_ResetHUDFonts(bool IsStarting)`
- Purpose: Reset interface fonts and optionally unload cached HUD textures during startup
- Inputs: `IsStarting` (true if startup initialization, false for runtime reload)
- Outputs/Return: void
- Side effects: Calls `OGL_Reset()` on 4 interface font objects; if `IsStarting && hud_pict_loaded`, deletes HUD textures
- Calls: `get_interface_font()`, `FontSpecifier::OGL_Reset()`, `glDeleteTextures()`

### HUD_OGL_Class::update_motion_sensor
- Signature: `void HUD_OGL_Class::update_motion_sensor(short time_elapsed)`
- Purpose: Conditionally update motion sensor display based on game options and enabled status
- Inputs: `time_elapsed` (time delta, or NONE to reset)
- Outputs/Return: void
- Side effects: Calls motion sensor update/reset functions
- Calls: `GET_GAME_OPTIONS()`, `reset_motion_sensor()`, `motion_sensor_scan()`
- Notes: Only updates if motion sensor is enabled in options and `MotionSensorActive` is true

### HUD_OGL_Class::DrawShape
- Signature: `void HUD_OGL_Class::DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- Purpose: Draw a bitmap shape clipped by source rectangle and scaled to destination rectangle
- Inputs: `shape` (descriptor), `dest` (target screen rect), `src` (source clipping rect within bitmap)
- Outputs/Return: void; renders textured quad
- Side effects: Enables GL_TEXTURE_2D, disables GL_BLEND, sets texture matrix
- Calls: `TextureManager::Setup()`, `SetupTextureMatrix()`, `RenderNormal()`, `RestoreTextureMatrix()`, GL immediate-mode drawing
- Notes: Computes UV scale/offset from source/dest ratio; skips rendering if TextureManager setup fails

### HUD_OGL_Class::DrawShapeAtXY
- Signature: `void HUD_OGL_Class::DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- Purpose: Draw a shape at fixed screen coordinates, with optional alpha blending for transparency
- Inputs: `shape`, `x`, `y` (top-left position), `transparency` (enable blend mode)
- Outputs/Return: void; renders textured quad
- Side effects: Enables GL_TEXTURE_2D; conditionally enables GL_BLEND with SRC_ALPHA/ONE_MINUS_SRC_ALPHA blend func
- Calls: TextureManager, GL drawing
- Notes: Uses full shape dimensions; blend mode suitable for semi-transparent HUD overlays

### HUD_OGL_Class::DrawTexture
- Signature: `void HUD_OGL_Class::DrawTexture(shape_descriptor texture, short x, short y, int size)`
- Purpose: Draw a wall texture at fixed position and size
- Inputs: `texture` (shape descriptor), `x`, `y` (position), `size` (square dimensions)
- Outputs/Return: void; renders textured quad
- Side effects: Enables GL_TEXTURE_2D, disables GL_BLEND
- Calls: TextureManager, GL drawing
- Notes: Ignores texture's native dimensions and uses `size` parameter (commented code shows alternative)

### HUD_OGL_Class::DrawText
- Signature: `void HUD_OGL_Class::DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- Purpose: Render colored text in a rectangle using interface font and color
- Inputs: `text` (string), `dest` (bounding rect), `flags` (alignment/wrap), `font_id` (interface font), `text_color` (interface color index)
- Outputs/Return: void; renders text
- Side effects: Sets GL color; delegates to font's OGL renderer
- Calls: `get_interface_color()`, `get_interface_font()`, `FontSpecifier::OGL_DrawText()`
- Notes: Converts 16-bit interface color to GL float RGB

### HUD_OGL_Class::FillRect
- Signature: `void HUD_OGL_Class::FillRect(screen_rectangle *r, short color_index)`
- Purpose: Fill a rectangle with solid color
- Inputs: `r` (rectangle), `color_index` (interface color)
- Outputs/Return: void; renders filled quad
- Side effects: Disables GL_TEXTURE_2D, sets GL color
- Calls: `get_interface_color()`, `glRecti()`

### HUD_OGL_Class::FrameRect
- Signature: `void HUD_OGL_Class::FrameRect(screen_rectangle *r, short color_index)`
- Purpose: Draw rectangle outline with 1-pixel line width
- Inputs: `r` (rectangle), `color_index` (interface color)
- Outputs/Return: void; renders line loop
- Side effects: Disables GL_TEXTURE_2D, sets line width
- Calls: GL line primitives
- Notes: Adds 0.5 offset to vertices to avoid pixel-center aliasing

### HUD_OGL_Class::SetClipPlane
- Signature: `void HUD_OGL_Class::SetClipPlane(int x, int y, int c_x, int c_y, int radius)`
- Purpose: Enable a clip plane tangent to a circle for rendering motion sensor blips with circular approximation
- Inputs: `x`, `y` (blip offset), `c_x`, `c_y` (circle center), `radius` (circle radius)
- Outputs/Return: void; enables GL_CLIP_PLANE0
- Side effects: Computes and sets clip plane equation
- Calls: `sqrt()`, `glClipPlane()`
- Notes: Skips if blip distance Γëñ 2.0; equation format: `{-nx, -ny, 0, nx┬╖tx + ny┬╖ty}`

### HUD_OGL_Class::DisableClipPlane
- Signature: `void HUD_OGL_Class::DisableClipPlane(void)`
- Purpose: Disable the clip plane
- Calls: `glDisable(GL_CLIP_PLANE0)`

### HUD_OGL_Class::draw_message_area
- Signature: `void HUD_OGL_Class::draw_message_area(short)`
- Purpose: Render the player name panel in the HUD message area
- Side effects: Calls shape and player-name rendering functions
- Calls: `DrawShapeAtXY()`, `draw_player_name()`
- Notes: Hardcoded message area offset; overrides virtual method from base class

## Control Flow Notes
Follows standard game frame pattern: `OGL_DrawHUD()` loads texture once (cached), then each frame marks dynamic displays dirty and calls `update_everything()` to redraw all HUD elements. Motion sensor updates conditionally; uses clip planes for circular blip rendering approximation.

## External Dependencies
- **FontHandler.h**: `FontSpecifier::OGL_DrawText()` for text rendering
- **game_window.h**: Dirty-marking functions (`mark_*_as_dirty()`); Mac HUD buffer pointer
- **images.h**: `get_picture_resource_from_images()`, `picture_to_surface()`
- **OpenGL**: Platform-specific includes (`<GL/gl.h>`)
- **SDL** (conditional): `SDL_Surface`, `SDL_BlitSurface()` for non-Mac rendering
- **MacOS APIs** (conditional): `GWorldPtr`, `PixMapHandle`, pixel-format conversion
- Externals: `HUD_Class` (base), `current_player_index`, `MotionSensorActive`, `LuaTexturePaletteSize()`, interface color/font accessors
