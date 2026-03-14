# Source_Files/RenderMain/Crosshairs_SDL.cpp

## File Purpose
SDL-based implementation for rendering game crosshairs. Provides state management (active/inactive) and rendering logic for two crosshair shapes: traditional plus-sign and circular (octagon approximation).

## Core Responsibilities
- Manage crosshairs active/inactive toggle state
- Render plus-sign crosshairs via four filled rectangles (left/right/top/bottom arms)
- Render circular crosshairs as octagon using 12 line segments (4 quadrants ├ù 3 segments each)
- Color mapping from game CrosshairData to SDL pixel format
- Center crosshairs on render surface

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| CrosshairData | struct | Holds color, thickness, distance-from-center, length, shape type, opacity (from Crosshairs.h) |
| world_point2d | struct | 2D coordinate pair (x, y) used for line segment endpoints |
| SDL_Surface | opaque type | SDL rendering surface target |
| RGBColor | struct | RGB color components (16-bit each) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| _Crosshairs_IsActive | bool | static | Tracks whether crosshairs are currently displayed |

## Key Functions / Methods

### Crosshairs_IsActive
- Signature: `bool Crosshairs_IsActive(void)`
- Purpose: Query current crosshairs active state
- Inputs: None
- Outputs/Return: bool (true if active, false otherwise)
- Side effects: None
- Calls: None
- Notes: Simple getter for static state

### Crosshairs_SetActive
- Signature: `bool Crosshairs_SetActive(bool NewState)`
- Purpose: Toggle crosshairs visibility
- Inputs: NewState (desired active state)
- Outputs/Return: bool (the new state)
- Side effects: Modifies _Crosshairs_IsActive
- Calls: None
- Notes: Assignment-based return

### Crosshairs_Render
- Signature: `bool Crosshairs_Render(SDL_Surface *s)`
- Purpose: Draw crosshairs centered in SDL surface (frame-level operation)
- Inputs: SDL_Surface pointer (render target)
- Outputs/Return: bool (true if drawn, false if inactive)
- Side effects: Modifies SDL surface pixel data via SDL_FillRect() and draw_line()
- Calls: GetCrosshairData(), SDL_MapRGB(), SDL_FillRect(), draw_line()
- Notes:
  - Early return if not active
  - Hardcoded center: `(w/2 - 1, h/2 - 1)`
  - **RealCrosshairs path**: draws 4 rectangles with reused SDL_Rect struct (mutating x/y/w/h between calls)
  - **Circle path**: precalculates 6 x-points and 6 y-points per axis; nested loop over 2├ù2 quadrants; each quadrant draws 3 line segments (vertical, diagonal, horizontal) using world_point2d endpoints
  - Circle is octagon approximation for OpenGL rendering convenience (per comment)

## Control Flow Notes
Called during per-frame render phase to composite HUD crosshair overlay. Reads current crosshair configuration (shape, color, dimensions, thickness) from preferences via `GetCrosshairData()`. No initialization or shutdown hooks visible.

## External Dependencies
- **SDL**: SDL_Surface, SDL_Rect, SDL_FillRect(), SDL_MapRGB()
- **Crosshairs.h**: CrosshairData struct, GetCrosshairData(), Crosshairs_IsActive(), Crosshairs_SetActive()
- **screen_drawing.h**: draw_line(SDL_Surface*, world_point2d*, world_point2d*, uint32, int)
- **world.h**: world_point2d struct
- **cseries.h**: uint32, int16, short type definitions
