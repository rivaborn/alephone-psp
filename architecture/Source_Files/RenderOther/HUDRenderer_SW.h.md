# Source_Files/RenderOther/HUDRenderer_SW.h

## File Purpose
Defines a software-rendered HUD renderer class (`HUD_SW_Class`) that inherits from the base `HUD_Class`. Provides implementations of abstract rendering methods using CPU-based graphics functions for drawing the player's heads-up display (motion sensor, ammo, energy, oxygen, etc.).

## Core Responsibilities
- Implement motion sensor rendering (update and render logic)
- Provide primitive drawing operations (shapes, text, rectangles, textures) via software rendering
- Override base class abstract methods for entity blip (radar dot) rendering
- Manage clipping regions for motion sensor viewport (though SetClipPlane/DisableClipPlane are no-ops)
- Interface between the high-level HUD update logic and low-level screen drawing functions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HUD_SW_Class` | class | Software-rendered HUD renderer; inherits from `HUD_Class` |

## Global / File-Static State
None.

## Key Functions / Methods

### update_motion_sensor
- Signature: `void update_motion_sensor(short time_elapsed)`
- Purpose: Update motion sensor state (blips, animation, etc.) for this frame
- Inputs: `time_elapsed` ΓÇô milliseconds since last update
- Outputs/Return: void
- Side effects: Modifies internal motion sensor state
- Calls: (implementation in `.cpp` file, not visible here)
- Notes: Pure virtual in base class; must be implemented in derived class

### render_motion_sensor
- Signature: `void render_motion_sensor(short time_elapsed)`
- Purpose: Draw the motion sensor display to screen
- Inputs: `time_elapsed` ΓÇô milliseconds since last update
- Outputs/Return: void
- Side effects: Writes to screen framebuffer via screen drawing functions
- Calls: (implementation in `.cpp` file)
- Notes: Pure virtual in base class

### draw_or_erase_unclipped_shape
- Signature: `void draw_or_erase_unclipped_shape(short x, short y, shape_descriptor shape, bool draw)`
- Purpose: Draw or erase a shape at screen coordinates
- Inputs: `x`, `y` ΓÇô screen position; `shape` ΓÇô shape ID; `draw` ΓÇô true to draw, false to erase
- Outputs/Return: void
- Calls: (implementation in `.cpp`)
- Notes: Pure virtual in base class

### DrawShape / DrawShapeAtXY / DrawText / FillRect / FrameRect / DrawTexture
- **DrawShape**: Copy shape with source/dest clipping rectangles
- **DrawShapeAtXY**: Draw shape at (x, y) with optional transparency
- **DrawText**: Render text string with font, color, and formatting flags
- **FillRect**: Fill rectangle with solid color
- **FrameRect**: Draw rectangle outline
- **DrawTexture**: Draw scaled texture at position
- Inputs: shape/text/color IDs and geometric parameters
- Outputs/Return: void (all write to framebuffer)
- Calls: (implementations in `.cpp` file; likely call screen_drawing.h functions)
- Notes: All pure virtual in base class; used by HUD core logic to render primitives

### SetClipPlane / DisableClipPlane
- Signature: `void SetClipPlane(int x, int y, int c_x, int c_y, int radius)` / `void DisableClipPlane(void)`
- Purpose: Configure/disable circular clipping region for motion sensor viewport
- Notes: Both are empty no-op implementations; no clipping applied in software renderer

### erase_entity_blip / draw_entity_blip
- Signature: `void erase_entity_blip(point2d *location, shape_descriptor shape)` / `void draw_entity_blip(point2d *location, shape_descriptor shape)`
- Purpose: Erase or draw a radar blip (entity indicator) at a 2D world location
- Inputs: `location` ΓÇô 2D world position; `shape` ΓÇô shape ID for the blip
- Outputs/Return: void
- Notes: Pure virtual in base class; called by motion sensor rendering logic

## Control Flow Notes
This class is instantiated when software-based HUD rendering is selected (as opposed to OpenGL or other renderers). The frame update flow is:
1. **Init**: Constructor (`HUD_SW_Class()`) called at startup
2. **Per-frame update**: `update_everything()` (from base class) calls `update_motion_sensor()`, `update_weapon_panel()`, etc.
3. **Rendering**: Base class calls the drawing methods (`DrawShape`, `FillRect`, `DrawText`, etc.) to composite HUD elements onto the screen
4. **Shutdown**: Destructor called on cleanup

The software renderer draws primitives directly to the screen framebuffer using functions from `screen_drawing.h`.

## External Dependencies
- **Includes**: `HUDRenderer.h` (base class and HUD types)
- **Base class**: `HUD_Class` (abstract interface with pure virtual drawing/update methods)
- **Types used but not defined here**: `shape_descriptor`, `screen_rectangle`, `point2d`, `short` (likely from cseries.h or platform headers)
- **Implicit dependency**: `screen_drawing.h` functions (called by the `.cpp` implementations to render primitives)
- **Platform conditionals**: Windows-specific `#undef DrawText` to avoid macro conflicts with WinAPI
