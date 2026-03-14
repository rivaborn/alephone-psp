# Source_Files/RenderOther/HUDRenderer_SW.cpp

## File Purpose
Software-based HUD renderer implementing shape, text, and UI element drawing for the Aleph One game engine. Provides SDL-based implementations of motion sensor updates, texture drawing with rotation/scaling, and basic primitive rendering (rectangles, text).

## Core Responsibilities
- Update and render motion sensor display with dynamic status checks
- Draw shapes to HUD buffer with optional source/destination clipping
- Handle texture drawing with 90┬░ rotation and arbitrary rescaling
- Render text strings with font and color styling
- Draw filled and outlined rectangles for UI elements
- Manage SDL surface conversions for multi-format pixel depth support (8, 16, 32-bit)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_SW_Class` | class | Software-based HUD renderer inheriting from `HUD_Class` |
| `shape_descriptor` | typedef | Opaque handle to a game shape resource |
| `screen_rectangle` | struct | Rectangle with screen coordinates (defined elsewhere) |
| `SDL_Surface` | struct | SDL pixel buffer with format metadata |
| `pixel8/pixel16/pixel32` | typedef | Typed pixel pointers for format-specific operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `HUD_Buffer` | `SDL_Surface*` | extern | Target render buffer for all HUD drawing operations |
| `MotionSensorActive` | `bool` | extern | Runtime flag controlling motion sensor rendering |
| `ForceUpdate` | (inherited) | class member | Signals HUD invalidation to parent class |

## Key Functions / Methods

### update_motion_sensor
- **Signature:** `void HUD_SW_Class::update_motion_sensor(short time_elapsed)`
- **Purpose:** Update motion sensor display if enabled and functional; redraw if sensor state changed.
- **Inputs:** `time_elapsed` ΓÇö elapsed milliseconds since last update (or `NONE` to reset sensor)
- **Outputs/Return:** None (modifies `ForceUpdate` flag; triggers redraw via `DrawShapeAtXY`)
- **Side effects:** Calls motion sensor scanning functions, redraws interface mount if sensor changed, flags for screen update
- **Calls:** `GET_GAME_OPTIONS()`, `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `DrawShapeAtXY()`
- **Notes:** Only executes if `MotionSensorActive` and `_motion_sensor_does_not_work` flag is not set in game options; early exit if disabled

### DrawShape
- **Signature:** `void HUD_SW_Class::DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)`
- **Purpose:** Delegate to screen-drawing backend for shape rasterization.
- **Inputs:** shape descriptor, destination and source rectangles (for clipping/scaling)
- **Outputs/Return:** None
- **Side effects:** Renders to `HUD_Buffer`
- **Calls:** `_draw_screen_shape()`
- **Notes:** Pure delegation; actual rendering implemented in screen_drawing module

### DrawShapeAtXY
- **Signature:** `void HUD_SW_Class::DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- **Purpose:** Draw shape at absolute screen coordinates.
- **Inputs:** shape, x/y pixel position, transparency flag (software rendering ignores)
- **Outputs/Return:** None
- **Side effects:** Renders to `HUD_Buffer`
- **Calls:** `_draw_screen_shape_at_x_y()`
- **Notes:** Transparency parameter exists for API compatibility with OpenGL backend but is unused here

### DrawTexture
- **Signature:** `void HUD_SW_Class::DrawTexture(shape_descriptor shape, short x, short y, int size)`
- **Purpose:** Draw a textured shape with rescaling and 90┬░ rotation applied.
- **Inputs:** shape descriptor, x/y screen position, output size (square)
- **Outputs/Return:** None
- **Side effects:** Allocates/frees multiple temporary `SDL_Surface` objects; memory churn for each call
- **Calls:** `get_shape_surface()`, `SDL_DisplayFormat()`, `rescale_surface()`, `rotate_surface()`, `SDL_BlitSurface()`, `SDL_FreeSurface()`
- **Notes:** Complex pipeline: source ΓåÆ display format conversion (for 8-bit palette CLUT mismatch) ΓåÆ rescale ΓåÆ rotate ΓåÆ blit. All intermediate surfaces freed before return; no leaks observed but allocation is expensive per-frame

### DrawText
- **Signature:** `void HUD_SW_Class::DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- **Purpose:** Render text string into rectangle.
- **Inputs:** text (C string), destination rectangle, alignment/style flags, font ID, color index
- **Outputs/Return:** None
- **Side effects:** Renders to `HUD_Buffer`
- **Calls:** `_draw_screen_text()`
- **Notes:** Pure delegation

### FillRect / FrameRect
- **Signature:** `void HUD_SW_Class::FillRect(screen_rectangle *r, short color_index)` / `void HUD_SW_Class::FrameRect(screen_rectangle *r, short color_index)`
- **Purpose:** Fill or outline a rectangle with indexed color.
- **Inputs:** rectangle, color palette index
- **Outputs/Return:** None
- **Side effects:** Renders to `HUD_Buffer`
- **Calls:** `_fill_rect()` / `_frame_rect()`

**Helper functions (static):**
- `rotate<T>()` ΓÇö template rotating pixel rows 90┬░ via transpose indexing
- `rotate_surface()` ΓÇö allocates new SDL surface and applies rotation for all pixel depths; handles palette cloning for indexed color

## Control Flow Notes
Fits into frame-based rendering loop: motion sensor updates during game tick phase (if elapsed time provided); shape/text/rect drawing during HUD composition phase. All operations target off-screen `HUD_Buffer`, which is presumably composited to final display elsewhere.

## External Dependencies
- **Includes:** `HUDRenderer.h` (base class), `images.h` (SDL and resource loading), `shell.h` (shape/screen functions)
- **SDL:** `SDL_Surface`, `SDL_Rect`, `SDL_CreateRGBSurface`, `SDL_DisplayFormat`, `SDL_BlitSurface`, `SDL_FreeSurface`, `SDL_SetColors`
- **Defined elsewhere:** `_draw_screen_shape[_at_x_y]()`, `_draw_screen_text()`, `_fill_rect()`, `_frame_rect()`, `get_shape_surface()`, `rescale_surface()`, `reset_motion_sensor()`, `motion_sensor_scan()`, `motion_sensor_has_changed()`, `get_interface_rectangle()`, `BUILD_DESCRIPTOR()`, `GET_GAME_OPTIONS()`
