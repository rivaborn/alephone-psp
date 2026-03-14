# Source_Files/RenderOther/HUDRenderer_OGL.h

## File Purpose
OpenGL-specific implementation of HUD rendering. Provides concrete drawing operations for the in-game heads-up display by inheriting from `HUD_Class` and overriding abstract rendering methods.

## Core Responsibilities
- Implement motion sensor display updates and rendering
- Draw 2D shapes, text, and textures to the HUD using OpenGL primitives
- Manage entity blips (radar contacts) on the motion sensor
- Handle clipping planes for circular HUD element boundaries (e.g., motion sensor)
- Provide primitive drawing operations (rectangles, filled regions, framed outlines)
- Render message and notification areas

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| HUD_OGL_Class | class | OpenGL-specific HUD renderer; inherits from `HUD_Class` and implements rendering virtual methods |

## Global / File-Static State
None.

## Key Functions / Methods

### update_motion_sensor
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor internal state each frame
- Inputs: `time_elapsed` ΓÇô elapsed ticks/milliseconds since last frame
- Outputs/Return: None
- Side effects: Modifies motion sensor state (blip positions, animations)
- Calls: Not visible in header
- Notes: Virtual override; called before render_motion_sensor

### render_motion_sensor
- Signature: `void render_motion_sensor(short time_elapsed)`
- Purpose: Render the motion sensor display each frame
- Inputs: `time_elapsed` ΓÇô elapsed ticks/milliseconds
- Outputs/Return: None
- Side effects: Draws to OpenGL framebuffer
- Calls: Likely invokes DrawShape, DrawText, draw_entity_blip, and clipping methods
- Notes: Virtual override; must be called after update_motion_sensor

### draw_entity_blip
- Signature: `void draw_entity_blip(point2d *location, shape_descriptor shape)`
- Purpose: Draw a single entity contact (ally/enemy) on the motion sensor
- Inputs: `location` ΓÇô 2D sensor/screen coordinates; `shape` ΓÇô texture/shape ID
- Outputs/Return: None
- Side effects: Draws to framebuffer
- Calls: Not visible in header
- Notes: Virtual override; pairs with erase_entity_blip (empty stub in this class)

### DrawShape, DrawShapeAtXY
- Signature: `void DrawShape(shape_descriptor shape, screen_rectangle *dest, screen_rectangle *src)` and `void DrawShapeAtXY(shape_descriptor shape, short x, short y, bool transparency = false)`
- Purpose: Draw a shape/texture with optional source/dest clipping and transparency
- Inputs: Shape ID, rectangles/coordinates, optional alpha blending flag
- Outputs/Return: None
- Side effects: Draw to OpenGL framebuffer
- Calls: Not visible in header
- Notes: Virtual overrides; DrawShape is a full blit operation, DrawShapeAtXY is a convenience wrapper

### DrawText
- Signature: `void DrawText(const char *text, screen_rectangle *dest, short flags, short font_id, short text_color)`
- Purpose: Render text string to screen
- Inputs: C string, bounding rectangle, formatting flags, font ID, color index
- Outputs/Return: None
- Side effects: Draws to framebuffer
- Calls: Not visible in header
- Notes: Virtual override; wraps font/text rendering

### FillRect, FrameRect
- Signature: `void FillRect(screen_rectangle *r, short color_index)` and `void FrameRect(screen_rectangle *r, short color_index)`
- Purpose: Fill or outline a rectangle with solid color
- Inputs: Rectangle, color index
- Outputs/Return: None
- Side effects: Draw to framebuffer
- Calls: Not visible in header
- Notes: Virtual overrides; primitive drawing operations

### DrawTexture
- Signature: `void DrawTexture(shape_descriptor texture, short x, short y, int size)`
- Purpose: Draw a texture at screen position with specified size
- Inputs: Texture ID, screen position (x, y), size scaling
- Outputs/Return: None
- Side effects: Draws to OpenGL framebuffer
- Calls: Not visible in header
- Notes: Virtual override; specialized texture rendering

### SetClipPlane, DisableClipPlane
- Signature: `void SetClipPlane(int x, int y, int c_x, int c_y, int radius)` and `void DisableClipPlane(void)`
- Purpose: Enable/disable scissor or stencil clipping to a circular region
- Inputs: SetClipPlane takes clip center (c_x, c_y) and radius; DisableClipPlane takes none
- Outputs/Return: None
- Side effects: Modify OpenGL scissor/stencil state
- Calls: Not visible in header
- Notes: Virtual overrides; confine HUD drawing (e.g., motion sensor blips) to circular boundaries

### draw_or_erase_unclipped_shape
- Signature: `void draw_or_erase_unclipped_shape(short x, short y, shape_descriptor shape, bool draw)`
- Purpose: Draw or erase a shape outside normal clipping regions
- Inputs: Position, shape ID, draw/erase flag
- Outputs/Return: None
- Side effects: Modifies framebuffer
- Calls: Not visible in header
- Notes: Virtual override; used for motion sensor elements that exceed clipping boundaries

### draw_message_area
- Signature: `virtual void draw_message_area(short)`
- Purpose: Render on-screen messages or notification overlays
- Inputs: Time-related parameter
- Outputs/Return: None
- Side effects: Draws to framebuffer
- Calls: Not visible in header
- Notes: Virtual override; called each frame as part of HUD rendering pass

## Control Flow Notes
Fits into per-frame HUD rendering: `update_everything()` (base class) orchestrates calls to update_motion_sensor, render_motion_sensor, and various Draw* methods. SetClipPlane/DisableClipPlane bracket motion sensor drawing to enforce circular boundary.

## External Dependencies
- `HUDRenderer.h` ΓÇô base class `HUD_Class`, shared constants (MOTION_SENSOR_SIDE_LENGTH, etc.)
- Game engine types: `shape_descriptor`, `screen_rectangle`, `point2d`, `interface_state_data`
- OpenGL API (actual calls in .cpp implementation, not visible in header)
