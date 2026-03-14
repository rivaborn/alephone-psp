# Source_Files/RenderOther/OverheadMap_OGL.cpp

## File Purpose
OpenGL implementation of an overhead map renderer for the Marathon/Aleph One game engine. Subclass of `OverheadMapClass` that renders the in-game map efficiently using vertex caching and batch drawing to avoid CPU bottlenecks at high resolutions.

## Core Responsibilities
- Initialize and configure OpenGL state for 2D orthographic map rendering
- Batch and cache polygon geometry for efficient multi-draw rendering
- Batch and cache line segments with dynamic color and width changes
- Render game entities (players, objects, monsters) as geometric primitives
- Render text annotations on the map with layout control
- Accumulate and render movement paths as line strips
- Manage color and pen-size state to minimize redundant state changes

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OverheadMap_OGL_Class` | class | Subclass implementing OpenGL rendering backend for overhead map |
| `PolygonCache` | member (`vector<unsigned short>`) | Vertex indices for batched polygon rendering |
| `LineCache` | member (`vector<unsigned short>`) | Vertex indices for batched line rendering |
| `PathPoints` | member (`vector<world_point2d>`) | Accumulated points for a single path |
| `SavedColor` | member (`rgb_color`) | Current color state to detect changes |
| `SavedPenSize` | member (`short`) | Current line width to detect changes |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ViewWidth`, `ViewHeight` | `extern short` | global | Map viewport dimensions (defined in OGL_Render.cpp) |
| `RenderContext` | `extern AGLContext` | global (Mac only) | Current AGL rendering context for font rasterization |

## Key Functions / Methods

### begin_overall
- **Signature:** `void begin_overall()`
- **Purpose:** Initialize OpenGL state and clear the map viewport for a new frame.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Disables depth testing, culling, alpha test, blend, texturing, fog; sets line width to 1; clears viewport to black polygon; modifies client state arrays.
- **Calls:** `glColor3f()`, `glBegin(GL_POLYGON)`, `glVertex2f()`, `glEnd()`, `glDisable()`, `glDisableClientState()`, `glLineWidth()`.
- **Notes:** Paints black background rectangle instead of using `glClear()` (commented-out code suggests cross-platform compatibility). Disables most GL features irrelevant to 2D map overlay.

### end_overall
- **Signature:** `void end_overall()`
- **Purpose:** Restore OpenGL state after map rendering completes.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Re-enables texture coordinate client state; resets line width to 1.
- **Calls:** `glEnableClientState()`, `glLineWidth()`.

### begin_polygons / end_polygons
- **Signature:** `void begin_polygons()`, `void end_polygons()`
- **Purpose:** Frame polygon rendering batch; prepare vertex array state and reset color.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Clears `PolygonCache` vector; initializes `SavedColor` to black; sets vertex pointer if `USE_VERTEX_ARRAYS`.
- **Calls:** `glVertexPointer()`, color initialization.
- **Notes:** `end_polygons()` flushes by calling `DrawCachedPolygons()`.

### draw_polygon
- **Signature:** `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- **Purpose:** Cache polygon vertices and indices; flush previous batch if color changes.
- **Inputs:** 
  - `vertex_count`: Number of vertices.
  - `vertices`: Array of vertex indices into the global vertex array.
  - `color`: Fill color for this polygon.
- **Outputs/Return:** None.
- **Side effects:** Appends to `PolygonCache` (if `USE_VERTEX_ARRAYS`) or calls `glBegin(GL_POLYGON)` immediately. Updates `SavedColor` if color differs.
- **Calls:** `ColorsEqual()`, `DrawCachedPolygons()`, `SetColor()`, `GetVertex()`.
- **Notes:** Converts polygons to triangle fans for vertex array rendering. Non-array path renders immediately.

### DrawCachedPolygons
- **Signature:** `void DrawCachedPolygons()`
- **Purpose:** Flush accumulated polygon indices to GPU via single `glDrawElements()` call.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Executes GPU draw call if cache is non-empty; clears cache.
- **Calls:** `glDrawElements()` (if `USE_VERTEX_ARRAYS`).
- **Notes:** Renders as `GL_TRIANGLES` (output of triangle-fan conversion).

### begin_lines / end_lines / DrawCachedLines
- **Signature:** `void begin_lines()`, `void end_lines()`, `void DrawCachedLines()`
- **Purpose:** Frame line rendering batch with dynamic color/width tracking.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Initializes `SavedPenSize` to 1; clears `LineCache`; flushes cache on width/color change.
- **Calls:** `glLineWidth()`, `glDrawElements()`, `SetColor()`.
- **Notes:** `end_lines()` calls `DrawCachedLines()` to flush.

### draw_line
- **Signature:** `void draw_line(short *vertices, rgb_color& color, short pen_size)`
- **Purpose:** Cache line endpoints; flush if color or pen size changes.
- **Inputs:**
  - `vertices`: Two-element array of vertex indices.
  - `color`: Line color.
  - `pen_size`: Line width in pixels.
- **Outputs/Return:** None.
- **Side effects:** Appends to `LineCache` (if `USE_VERTEX_ARRAYS`). Updates `SavedColor` or `SavedPenSize` if changed.
- **Calls:** `ColorsEqual()`, `DrawCachedLines()`, `SetColor()`, `glLineWidth()`.
- **Notes:** Non-array path renders immediately as `GL_LINES`.

### draw_thing
- **Signature:** `void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- **Purpose:** Render a game object (rectangle or octagon circle) at a given location and scale.
- **Inputs:**
  - `center`: World location.
  - `color`: Fill/outline color.
  - `shape`: `_rectangle_thing` or `_circle_thing`.
  - `radius`: Size in world units.
- **Outputs/Return:** None.
- **Side effects:** Pushes/pops modelview matrix; translates and scales to object position.
- **Calls:** `SetColor()`, `glMatrixMode()`, `glPushMatrix()`, `glTranslatef()`, `glScalef()`, `glVertexPointer()`, `glDrawArrays()`, `glPopMatrix()`, `glLineWidth()`.
- **Notes:** Circle is approximated as an 8-vertex octagon. Uses vertex arrays with locally-defined shape data.

### draw_player
- **Signature:** `void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- **Purpose:** Render player character as a triangle with direction indicator (front point).
- **Inputs:**
  - `center`: Player location.
  - `facing`: Facing angle (0ΓÇô`FULL_CIRCLE` range).
  - `color`: Fill color.
  - `shrink`: Shrink factor (power-of-2 exponent for scaling).
  - `front`, `rear`: Triangle dimension parameters (in world units).
  - `rear_theta`: Angle offset for rear vertices (radians-like).
- **Outputs/Return:** None.
- **Side effects:** Builds triangle shape dynamically; pushes/pops modelview; applies translation, rotation, and scale.
- **Calls:** `SetColor()`, trigonometry, matrix ops, `glVertexPointer()`, `glDrawArrays()`.
- **Notes:** Draws both filled polygon and perimeter line (dual-draw for visibility at small scales).

### draw_text
- **Signature:** `void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- **Purpose:** Render text annotation on the map.
- **Inputs:**
  - `location`: Bottom-left anchor point.
  - `color`: Text color.
  - `text`: C-string to render.
  - `FontData`: Font metrics and rendering interface.
  - `justify`: `_justify_left` (0) or `_justify_center` (1).
- **Outputs/Return:** None.
- **Side effects:** Adjusts location for center justification; pushes/pops modelview; calls font renderer.
- **Calls:** `FontData.TextWidth()`, `SetColor()`, `FontData.OGL_Render()`, matrix ops.
- **Notes:** Font rendering is delegated to `FontSpecifier` (likely Mac-specific or platform-abstracted).

### set_path_drawing / draw_path / finish_path
- **Signature:** `void set_path_drawing(rgb_color& color)`, `void draw_path(short step, world_point2d &location)`, `void finish_path()`
- **Purpose:** Accumulate path points and render as a line strip.
- **Inputs:** 
  - `set_path_drawing`: color for the path.
  - `draw_path`: step number (0 to reset) and next point.
- **Outputs/Return:** None.
- **Side effects:** Clears `PathPoints` when `step <= 0`; appends points; renders on `finish_path()`.
- **Calls:** `SetColor()`, `glLineWidth()`, `glVertexPointer()`, `glDrawArrays()`.
- **Notes:** Renders the entire accumulated path as a single `GL_LINE_STRIP` for efficiency.

## Control Flow Notes
Map rendering follows a strict begin/end lifecycle:
1. **Initialization:** `begin_overall()` clears viewport and configures OpenGL.
2. **Batched rendering:** Polygons (`begin_polygons()` ΓåÆ `draw_polygon()*` ΓåÆ `end_polygons()`) and lines (`begin_lines()` ΓåÆ `draw_line()*` ΓåÆ `end_lines()`).
3. **Immediate rendering:** `draw_thing()`, `draw_player()`, `draw_text()` render directly (not batched).
4. **Path rendering:** `set_path_drawing()` ΓåÆ `draw_path()*` ΓåÆ `finish_path()` renders a single line strip.
5. **Cleanup:** `end_overall()` restores state.

The batching strategy defers `glDrawElements()` calls until color or pen-size changes, reducing draw call overhead.

## External Dependencies
- **OpenGL headers:** `<GL/gl.h>`, `<GL/glu.h>` (platform-conditional includes for macOS, Windows, Linux).
- **AGL (macOS only):** `<AGL/agl.h>` for font context.
- **cseries.h:** Core utility macros and types.
- **OverheadMap_OGL.h:** Class definition and member declarations.
- **map.h:** World geometry structures (`world_point2d`, `world_point3d`, `rgb_color`, `angle`), constants (`FULL_CIRCLE`), and global vertex/endpoint arrays (via `GetVertex()`, `GetVertexStride()`, `GetFirstVertex()` macros/functions not defined in this file).

**Defined elsewhere (inferred):**
- `SetColor()`, `ColorsEqual()`: Utility inlines in this file.
- `GetVertex()`, `GetVertexStride()`, `GetFirstVertex()`: Likely defined in parent class or utility header.
