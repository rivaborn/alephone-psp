# Source_Files/RenderMain/Rasterizer.h

## File Purpose
Abstract base class defining the interface for rasterizer implementations (software, OpenGL, etc.). Serves as the primary abstraction layer for rendering operations within the game's 3D/2D rendering pipeline. Subclasses provide concrete implementations for different rendering backends.

## Core Responsibilities
- Define the virtual interface for view setup and perspective configuration
- Manage rendering lifecycle (Begin/End boundaries)
- Provide texture rasterization for polygons (horizontal and vertical) and rectangles
- Handle foreground object rendering (weapons in hand) with optional reflection
- Allow backend-specific implementations to override all operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `RasterizerClass` | class | Abstract base class for all rasterizer implementations |

## Global / File-Static State
None.

## Key Functions / Methods

### SetView
- Signature: `virtual void SetView(view_data& View)`
- Purpose: Configure the rasterizer with view parameters (projection, camera position, FOV, etc.)
- Inputs: Reference to `view_data` struct containing camera and projection state
- Outputs/Return: None
- Side effects: Updates internal rasterizer state
- Notes: Must be called before any rendering operations; empty default implementation

### SetForeground
- Signature: `virtual void SetForeground()`
- Purpose: Switch the rasterizer into foreground rendering mode for HUD elements (weapons, hands)
- Inputs: None
- Outputs/Return: None
- Side effects: Changes rendering mode/state
- Notes: Typically called after main scene rendering; empty default

### SetForegroundView
- Signature: `virtual void SetForegroundView(bool HorizReflect)`
- Purpose: Configure perspective for foreground objects with optional horizontal mirroring
- Inputs: `HorizReflect` ΓÇö boolean flag for left/right reflection
- Outputs/Return: None
- Side effects: Updates view transform
- Notes: Used for rendering reflected/mirrored weapon views

### Begin / End
- Signature: `virtual void Begin()` and `virtual void End()`
- Purpose: Delimit a rendering batch; Begin prepares the backend, End finalizes
- Inputs: None
- Outputs/Return: None
- Side effects: State changes (e.g., buffer binding, state validation)
- Notes: Empty defaults; subclasses use these to bracket rendering operations

### texture_horizontal_polygon / texture_vertical_polygon / texture_rectangle
- Signature: Three overloads taking `polygon_definition&` or `rectangle_definition&`
- Purpose: Rasterize textured geometry; horizontal and vertical polygons handle walls/floors, rectangles handle sprites
- Inputs: Reference to polygon or rectangle definition (geometry + texture data)
- Outputs/Return: None
- Side effects: Modifies framebuffer
- Notes: Empty defaults; actual rendering delegated to subclass implementations

## Control Flow Notes
Typical usage pattern:
1. `SetView()` ΓÇö configure camera/projection once per frame
2. `Begin()` ΓÇö initialize rendering backend
3. Multiple `texture_*()` calls to draw scene geometry
4. `SetForeground()` / `SetForegroundView()` ΓÇö switch to foreground layer (weapons)
5. Additional `texture_*()` calls for HUD elements
6. `End()` ΓÇö finalize rendering and present

## External Dependencies
- `#include "render.h"` ΓÇö provides `view_data`, `polygon_definition`, `rectangle_definition`, `bitmap_definition`
- `#include "OGL_Render.h"` (conditional `HAVE_OPENGL`) ΓÇö OpenGL-specific definitions, included but not used in this header
- Standard C++ virtual method mechanism
