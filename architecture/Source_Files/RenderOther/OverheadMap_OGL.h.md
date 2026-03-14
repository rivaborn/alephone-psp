# Source_Files/RenderOther/OverheadMap_OGL.h

## File Purpose
Defines `OverheadMap_OGL_Class`, an OpenGL-specific subclass of `OverheadMapClass` for rendering the overhead/automap in-game HUD. Provides graphics-API-specific implementations of map rendering operations including polygons, lines, entities, and text with geometry caching for batched drawing.

## Core Responsibilities
- Override virtual rendering methods from `OverheadMapClass` with OpenGL implementations
- Manage rendering lifecycle through begin/end pairs for polygons, lines, and the overall render frame
- Cache polygon and line geometry for batched rendering
- Draw map primitives: polygons (terrain), lines (walls/elevation), things (items/monsters), player indicator
- Support text rendering with font and justification settings
- Buffer path points for visualizing entity/monster movement trails

## Key Types / Data Structures
None defined in this file (uses types from parent class and included headers).

## Global / File-Static State
None.

## Key Functions / Methods

### begin_overall / end_overall
- Signature: `void begin_overall()`, `void end_overall()`
- Purpose: Mark start and end of entire overhead map render frame
- Inputs: None
- Outputs/Return: None
- Side effects: OpenGL state setup/teardown (inferred)
- Calls: Virtual overrides
- Notes: Called once per frame, wraps all other draw calls

### begin_polygons / draw_polygon / end_polygons
- Signature: `void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- Purpose: Render map polygons (terrain floors)
- Inputs: Vertex count, vertex indices, polygon color
- Outputs/Return: None
- Side effects: Accumulates geometry in `PolygonCache`; stores color in `SavedColor`
- Calls: Virtual overrides
- Notes: Batches polygon data; `DrawCachedPolygons()` flushes to GPU

### begin_lines / draw_line / end_lines
- Signature: `void draw_line(short *vertices, rgb_color& color, short pen_size)`
- Purpose: Render map line segments (walls, elevation boundaries)
- Inputs: Vertex endpoints, color, pen width
- Outputs/Return: None
- Side effects: Accumulates geometry in `LineCache`; stores width in `SavedPenSize`
- Calls: Virtual overrides
- Notes: `end_lines()` flushes cached data via `DrawCachedLines()`

### draw_thing
- Signature: `void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- Purpose: Render a map object (item, monster, projectile, checkpoint)
- Inputs: Position, color, shape code, radius
- Outputs/Return: None
- Side effects: Immediate OpenGL draw (not cached)

### draw_player
- Signature: `void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Draw player indicator with directional pointer
- Inputs: Center position, facing angle, color, shrink factor, shape dimensions
- Outputs/Return: None
- Side effects: Immediate OpenGL draw

### draw_text
- Signature: `void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text on the overhead map (labels, annotations)
- Inputs: Screen position, color, text string, font, justification (0=left, 1=center)
- Outputs/Return: None
- Side effects: Immediate font rendering

### set_path_drawing / draw_path / finish_path
- Signature: `void draw_path(short step, world_point2d& location)`
- Purpose: Accumulate and visualize a path (monster movement trail)
- Inputs: Step index (0=first point), location
- Outputs/Return: None
- Side effects: Accumulates points in `PathPoints`; `finish_path()` flushes visualization
- Notes: `set_path_drawing()` configures line color before drawing

## Control Flow Notes
Typical render sequence:
1. `begin_overall()` ΓÇô initialize frame
2. `begin_polygons()` ΓåÆ `draw_polygon()` calls (batched) ΓåÆ `end_polygons()` ΓåÆ `DrawCachedPolygons()`
3. `begin_lines()` ΓåÆ `draw_line()` calls (batched) ΓåÆ `end_lines()` ΓåÆ `DrawCachedLines()`
4. Direct calls: `draw_thing()`, `draw_player()`, `draw_text()`
5. Path rendering: `set_path_drawing()` ΓåÆ `draw_path()` calls ΓåÆ `finish_path()`
6. `end_overall()` ΓÇô finalize frame

Caching pattern reduces OpenGL state changes and draw calls.

## External Dependencies
- `#include <vector>` ΓÇô STL for geometry caches
- `#include "OverheadMapRenderer.h"` ΓÇô parent class `OverheadMapClass`, types (`rgb_color`, `world_point2d`, `FontSpecifier`, `angle`)
- External symbols: `rgb_color`, `world_point2d`, `angle`, `FontSpecifier` (defined elsewhere in engine)
