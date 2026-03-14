# Source_Files/RenderMain/Rasterizer_OGL.h

## File Purpose
OpenGL implementation of the rasterizer interface. Provides a concrete class that delegates rendering operations to underlying OpenGL (OGL_*) functions while maintaining the abstract rasterizer API contract.

## Core Responsibilities
- Implement the abstract `RasterizerClass` interface for OpenGL-based rendering
- Manage view and projection state for OpenGL rendering context
- Delegate rendering of textured polygons and rectangles to OGL functions
- Support foreground rendering for UI elements (weapons in hand, etc.)
- Provide frame delimiters (Begin/End) for OpenGL rendering passes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Rasterizer_OGL_Class | class | OpenGL rasterizer implementation; inherits from RasterizerClass |

## Global / File-Static State
None.

## Key Functions / Methods

### SetView
- Signature: `void SetView(view_data& View)`
- Purpose: Configure the rasterizer's view parameters before rendering
- Inputs: `View` ΓÇö view data (camera position, orientation)
- Outputs/Return: void
- Side effects: Calls `OGL_SetView()` to configure OpenGL projection/view matrices
- Calls: `OGL_SetView()`
- Notes: Must be called before any rendering operations

### SetForeground
- Signature: `virtual void SetForeground()`
- Purpose: Switch rasterizer mode to render foreground objects (weapons, HUD elements)
- Inputs: none
- Outputs/Return: void
- Side effects: Calls `OGL_SetForeground()` to adjust rendering state
- Calls: `OGL_SetForeground()`

### SetForegroundView
- Signature: `virtual void SetForegroundView(bool HorizReflect)`
- Purpose: Configure view transformation for foreground objects with optional mirroring
- Inputs: `HorizReflect` ΓÇö if true, apply horizontal reflection
- Outputs/Return: void
- Side effects: Calls `OGL_SetForegroundView()` to apply view transformation
- Calls: `OGL_SetForegroundView()`

### Begin / End
- Signature: `void Begin()`, `void End()`
- Purpose: Frame delimiters for OpenGL rendering pass
- Inputs: none
- Outputs/Return: void
- Side effects: `Begin()` calls `OGL_StartMain()`; `End()` calls `OGL_EndMain()`
- Calls: `OGL_StartMain()`, `OGL_EndMain()`
- Notes: Should be paired; Begin must precede all rendering calls in a frame

### texture_horizontal_polygon / texture_vertical_polygon
- Signature: `void texture_horizontal_polygon(polygon_definition& textured_polygon)`, `void texture_vertical_polygon(...)`
- Purpose: Render textured polygons with orientation control (horizontal/vertical orientation)
- Inputs: `textured_polygon` ΓÇö polygon definition including vertices and texture info
- Outputs/Return: void
- Side effects: Calls `OGL_RenderWall(textured_polygon, false/true)`
- Calls: `OGL_RenderWall()`
- Notes: Second parameter indicates orientation; likely used for walls

### texture_rectangle
- Signature: `void texture_rectangle(rectangle_definition& textured_rectangle)`
- Purpose: Render a textured rectangle (sprite)
- Inputs: `textured_rectangle` ΓÇö rectangle definition with texture
- Outputs/Return: void
- Side effects: Calls `OGL_RenderSprite()`
- Calls: `OGL_RenderSprite()`
- Notes: Used for sprites or flat UI elements

## Control Flow Notes
This class acts as a facade/adapter in the render pipeline: `SetView()` ΓåÆ `Begin()` ΓåÆ rendering calls (`texture_*`) ΓåÆ optional `SetForeground()` section ΓåÆ `End()`. Whether multiple foreground passes occur per frame is not inferable from this file.

## External Dependencies
- **Base class**: `RasterizerClass` (Rasterizer.h) ΓÇö abstract rendering interface
- **OpenGL functions** (defined elsewhere): `OGL_SetView()`, `OGL_SetForeground()`, `OGL_SetForegroundView()`, `OGL_StartMain()`, `OGL_EndMain()`, `OGL_RenderWall()`, `OGL_RenderSprite()`
- **Data structures** (defined elsewhere): `view_data`, `polygon_definition`, `rectangle_definition`
