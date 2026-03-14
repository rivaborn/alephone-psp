# Source_Files/RenderOther/OverheadMap_SDL.h

## File Purpose
SDL-specific implementation of the overhead map renderer. This class is a concrete subclass of `OverheadMapClass` that implements all virtual rendering methods using SDL graphics primitives for drawing the in-game mini-map.

## Core Responsibilities
- Override base class virtual methods to render map elements using SDL
- Draw polygons (terrain/level geometry) with colors
- Draw lines (walls/elevation changes) with variable pen sizes
- Draw things (entities: monsters, items, projectiles) as shapes
- Draw the player indicator with directional orientation
- Render text annotations for map labels
- Manage path visualization for checkpoint navigation

## Key Types / Data Structures
None (uses types from base class).

## Global / File-Static State
None.

## Key Functions / Methods

### draw_polygon
- Signature: `void draw_polygon(short vertex_count, short *vertices, rgb_color &color)`
- Purpose: Render a filled polygon representing map geometry
- Inputs: vertex count, array of vertex indices, fill color
- Outputs/Return: None (void)
- Side effects: Renders to SDL surface
- Calls: Pure virtual override; implementation defined in `.cpp`
- Notes: Base class provides wrapper to look up color from config

### draw_line
- Signature: `void draw_line(short *vertices, rgb_color &color, short pen_size)`
- Purpose: Draw a line segment on the map (wall, elevation change, control panel)
- Inputs: endpoint vertex indices, line color, pen width
- Outputs/Return: None
- Side effects: Renders to SDL surface
- Calls: Pure virtual override
- Notes: Pen size scales with zoom level; configurable styles per line type

### draw_thing
- Signature: `void draw_thing(world_point2d &center, rgb_color &color, short shape, short radius)`
- Purpose: Render an entity (monster, item, projectile, checkpoint)
- Inputs: center point, color, shape constant (`_rectangle_thing` or `_circle_thing`), radius
- Outputs/Return: None
- Side effects: Renders to SDL surface
- Calls: Pure virtual override
- Notes: Shape and radius are scale-dependent; configurable per entity type

### draw_player
- Signature: `void draw_player(world_point2d &center, angle facing, rgb_color &color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render player indicator with directional orientation marker
- Inputs: map position, facing angle, color, size parameters (shrink value), front/rear arc dimensions
- Outputs/Return: None
- Side effects: Renders to SDL surface
- Calls: Pure virtual override
- Notes: Shrink is computed from scale; front/rear define a directional wedge/arc

### draw_text
- Signature: `void draw_text(world_point2d &location, rgb_color &color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text labels on the map (map name, annotations)
- Inputs: screen/map location, text color, text string, font specification, justification (`_justify_left`, `_justify_center`)
- Outputs/Return: None
- Side effects: Renders text to SDL surface
- Calls: Pure virtual override
- Notes: Font is abstracted via `FontSpecifier` for portability

### set_path_drawing
- Signature: `void set_path_drawing(rgb_color &color)`
- Purpose: Initialize state for rendering the player's path trace
- Inputs: path color
- Outputs/Return: None
- Side effects: Sets `path_pixel` and initial `path_point`
- Calls: Pure virtual override
- Notes: Called once before a sequence of `draw_path()` calls

### draw_path
- Signature: `void draw_path(short step, world_point2d &location)`
- Purpose: Draw a single point along the recorded player path
- Inputs: step number (0 = first point), world coordinates
- Outputs/Return: None
- Side effects: Renders line segment or point; updates `path_point` state
- Calls: Pure virtual override
- Notes: Connected sequentially to form a trace line; state managed via `path_pixel` and `path_point`

## Control Flow Notes
This class is instantiated and called by the game engine during the UI/HUD rendering phase. The base class `Render()` method drives the rendering pipeline, invoking virtual methods in sequence: `begin_*()` / draw calls / `end_*()` groups for polygons, lines, things, and paths. SDL rendering is synchronous; output targets the current SDL surface.

## External Dependencies
- `OverheadMapRenderer.h` ΓÇö base class definition; includes game types (`rgb_color`, `world_point2d`, `angle`), font abstraction (`FontSpecifier`), and configuration data structures
- SDL library ΓÇö graphics rendering backend (not shown in header; implementation in `.cpp`)
- Standard game types ΓÇö `world_point2d`, `angle`, `rgb_color` (defined elsewhere)
