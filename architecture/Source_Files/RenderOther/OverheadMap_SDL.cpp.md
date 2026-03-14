# Source_Files/RenderOther/OverheadMap_SDL.cpp

## File Purpose
SDL-based implementation of the overhead map renderer for the Aleph One game engine. Provides concrete drawing implementations for map visualization, rendering polygons, lines, objects, players, text, and movement paths to an SDL surface.

## Core Responsibilities
- Render filled polygons (map regions) with color
- Render lines (walls, grid) with variable pen width
- Render map objects (scenery, items) as rectangles or octagonal circles
- Render player position and facing direction as a triangle
- Render text annotations on the map
- Maintain and render player movement path traces

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `world_point2d` | typedef | 2D world coordinates for map positions |
| `rgb_color` | struct | RGB color with 16-bit components (red, green, blue) |
| `angle` | typedef | Angular direction/facing in game units |
| `SDL_Surface` | SDL struct | SDL rendering surface target |
| `SDL_Rect` | SDL struct | Rectangle region for SDL operations |
| `FontSpecifier` | class | Font configuration (family, style) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `draw_surface` | `SDL_Surface*` | extern (from screen_sdl.cpp) | Active SDL surface for all drawing operations |
| `vertex_array` | `world_point2d*` (static) | function-static in `draw_polygon()` | Reusable buffer for polygon vertex coordinates |
| `max_vertices` | `int` (static) | function-static in `draw_polygon()` | Tracks allocated vertex buffer size |
| `path_pixel` | `uint32` | class member (private) | Cached SDL pixel color for current path |
| `path_point` | `world_point2d` | class member (private) | Last recorded path point for line drawing |

## Key Functions / Methods

### draw_polygon
- **Signature:** `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- **Purpose:** Render a filled polygon (map region) on the overhead map
- **Inputs:** vertex count, array of vertex indices, fill color
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Allocates/reallocates static `vertex_array` if needed; modifies SDL surface
- **Calls:** `GetVertex()` (parent class), `SDL_MapRGB()`, `::draw_polygon()` (screen_drawing.h)
- **Notes:** Uses dynamic vertex buffer to avoid per-frame allocation. Color is shifted right by 8 bits (16-bit ΓåÆ 8-bit per channel).

### draw_line
- **Signature:** `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- **Purpose:** Render a line segment (wall, boundary) with adjustable width
- **Inputs:** array of two vertex indices, color, pen width in pixels
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Modifies SDL surface
- **Calls:** `GetVertex()` (parent), `SDL_MapRGB()`, `::draw_line()` (screen_drawing.h)
- **Notes:** Pen size directly controls line thickness.

### draw_thing
- **Signature:** `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- **Purpose:** Render a map object (item, scenery) as either a filled rectangle or octagon circle
- **Inputs:** center position, color, shape type (`_rectangle_thing` or `_circle_thing`), bounding radius
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Modifies SDL surface; allocates temporary circle vertex array on stack
- **Calls:** `SDL_MapRGB()`, `SDL_FillRect()`, `::draw_line()` (for circle edges)
- **Notes:** Circle is approximated as 8-point octagon. Radius scaling uses 0.75x and 1.5x factors for centering adjustment.

### draw_player
- **Signature:** `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- **Purpose:** Render player position and facing direction as a filled triangle
- **Inputs:** center position, facing angle, color, shrink divisor, front/rear distances, rear angle spread
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Modifies SDL surface
- **Calls:** `SDL_MapRGB()`, `translate_point2d()` (geometry), `normalize_angle()`, `::draw_polygon()`
- **Notes:** Triangle points computed from center + translated offsets. Front point extends forward along facing; two rear points form base.

### draw_text
- **Signature:** `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- **Purpose:** Render text annotation on the overhead map
- **Inputs:** location, color, text string, font specification, justification flag
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Modifies SDL surface
- **Calls:** `SDL_MapRGB()`, `text_width()`, `::draw_text()` (screen_drawing.h)
- **Notes:** Supports center justification by computing text width offset.

### set_path_drawing
- **Signature:** `void set_path_drawing(rgb_color &color)`
- **Purpose:** Cache the color for subsequent path line rendering
- **Inputs:** color for path display
- **Outputs/Return:** None
- **Side effects:** Updates `path_pixel` member variable
- **Calls:** `SDL_MapRGB()`
- **Notes:** Must be called before `draw_path()` to establish the rendering color.

### draw_path
- **Signature:** `void draw_path(short step, world_point2d &location)`
- **Purpose:** Render a path trace connecting consecutive player positions
- **Inputs:** step counter (0 = first point, nonzero = draw line), current location
- **Outputs/Return:** None (side effect: draws to `draw_surface`)
- **Side effects:** Updates `path_point`; modifies SDL surface if step > 0
- **Calls:** `::draw_line()` (screen_drawing.h)
- **Notes:** `step == 0` initializes without drawing; `step > 0` draws from previous point. Invariant: `path_point` always holds the most recent location.

## Control Flow Notes
This file implements the rendering layer for the overhead map, called during the frame's map display phase. Methods are invoked by parent class `OverheadMapClass` to render individual geometry/object elements. No explicit frame/update cycle; drawing occurs on demand during the UI/HUD render pass. The `static vertex_array` in `draw_polygon()` persists across frames for performance.

## External Dependencies
- **SDL library** (`SDL.h` via cseries.h): surface operations, color mapping, rectangle filling
- **Parent class** `OverheadMapClass` (from OverheadMapRenderer.h): defines interface; `GetVertex()` accessor
- **Map module** (`map.h`): geometry types (`world_point2d`, `world_distance`, `angle`)
- **Screen drawing** (`screen_drawing.h`): low-level SDL drawing functions (`draw_polygon`, `draw_line`, `draw_text`); font handling
- **Global state** (`screen_sdl.cpp`): `draw_surface` SDL_Surface pointer
