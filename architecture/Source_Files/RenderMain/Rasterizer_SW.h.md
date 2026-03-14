# Source_Files/RenderMain/Rasterizer_SW.h

## File Purpose
Defines the software rasterizer implementation class (`Rasterizer_SW_Class`) that inherits from the base `RasterizerClass`. Acts as a thin wrapper layer for software-based polygon and rectangle rasterization, delegating actual rendering logic to the "scottish_textures.c" module.

## Core Responsibilities
- Provide a concrete software rasterizer implementation for the engine's rendering pipeline
- Maintain pointers to view data and screen bitmap used by the texture rasterization system
- Declare the interface for rendering textured horizontal polygons, vertical polygons, and rectangles
- Serve as the abstraction layer between the game engine's renderer and low-level software rasterization code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Rasterizer_SW_Class` | class | Software rasterizer that plugs into the engine's rendering abstraction |

## Global / File-Static State
None.

## Key Functions / Methods

### SetView
- Signature: `void SetView(view_data& View)`
- Purpose: Configures the rasterizer with the current view data before rendering begins
- Inputs: Reference to `view_data` structure containing camera/view configuration
- Outputs/Return: None
- Side effects: Updates the internal `view` member pointer
- Calls: None
- Notes: Marked as a requirement in comments ("be sure to call it before doing any rendering")

### texture_horizontal_polygon
- Signature: `void texture_horizontal_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a textured polygon with horizontal orientation
- Inputs: Reference to polygon definition (geometry, texture, material data)
- Outputs/Return: Writes to screen bitmap
- Side effects: Modifies screen/framebuffer via rasterization
- Calls: Implementation defined in `scottish_textures.c`
- Notes: Virtual method override; actual implementation not visible in this file

### texture_vertical_polygon
- Signature: `void texture_vertical_polygon(polygon_definition& textured_polygon)`
- Purpose: Rasterize a textured polygon with vertical orientation
- Inputs: Reference to polygon definition
- Outputs/Return: Writes to screen bitmap
- Side effects: Modifies screen/framebuffer via rasterization
- Calls: Implementation defined in `scottish_textures.c`
- Notes: Virtual method override; likely an optimization for vertical vs. horizontal orientation

### texture_rectangle
- Signature: `void texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Rasterize a textured rectangular region
- Inputs: Reference to rectangle definition
- Outputs/Return: Writes to screen bitmap
- Side effects: Modifies screen/framebuffer via rasterization
- Calls: Implementation defined in `scottish_textures.c`
- Notes: Specialized rendering path for axis-aligned rectangles

## Control Flow Notes
This class is instantiated as a concrete implementation of the engine's rasterizer abstraction. It participates in the frame loop as follows:
1. **Setup phase**: Engine calls `SetView()` with current camera/viewport
2. **Render phase**: Engine calls texture rendering methods (`texture_horizontal_polygon`, `texture_vertical_polygon`, `texture_rectangle`) for each drawable surface
3. **Output**: Methods write pixel data directly to the screen bitmap

Not inferable from this file whether this class is used alongside `Begin()`/`End()` batch calls from the base class.

## External Dependencies
- **Includes**: `Rasterizer.h` (base class definition)
- **Defined elsewhere**: 
  - `view_data` (render.h or included via Rasterizer.h)
  - `bitmap_definition` (render.h or included via Rasterizer.h)
  - `polygon_definition` (render.h or included via Rasterizer.h)
  - `rectangle_definition` (render.h or included via Rasterizer.h)
  - `RasterizerClass` (Rasterizer.h)
  - Implementation of texture rendering methods (scottish_textures.c)
