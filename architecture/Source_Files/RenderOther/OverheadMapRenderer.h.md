# Source_Files/RenderOther/OverheadMapRenderer.h

## File Purpose
Defines the base class and configuration structures for rendering the overhead (top-down) map in the Aleph One game engine. Provides virtual render interface for subclasses to implement graphics-API-specific drawing (e.g., OpenGL, software rasterization).

## Core Responsibilities
- Define configuration data for overhead map appearance (colors, fonts, shapes, scales)
- Declare virtual interface for rendering polygons, lines, entities, annotations, and paths
- Manage coordinate transformation for viewport-relative map rendering
- Support automap generation including "false" automaps for checkpoint-based discovery
- Provide template-method orchestration of rendering passes (begin/draw/end stages)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `line_definition` | struct | Color and per-scale pen sizes for map lines (solid, elevation, control panel) |
| `thing_definition` | struct | Color, shape, and per-scale radii for drawable entities (civilians, monsters, items, projectiles, checkpoints) |
| `entity_definition` | struct | Player entity appearance: front/rear/rear_theta angles |
| `annotation_definition` | struct | Font and color for map annotations across different zoom scales |
| `map_name_definition` | struct | Map title rendering: color, font, vertical offset |
| `OvhdMap_CfgDataStruct` | struct | Master configuration container: polygon colors, line/thing/annotation/map-name definitions, entity shapes, visibility flags |
| `OverheadMapClass` | class | Base renderer; subclassed for specific graphics backends |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `saved_automap_lines` | `byte*` | private member | Temporary storage for original automap line state during false-automap generation (checkpoint mode) |
| `saved_automap_polygons` | `byte*` | private member | Temporary storage for original automap polygon state during false-automap generation |
| `ConfigPtr` | `OvhdMap_CfgDataStruct*` | public member | Pointer to configuration data; must be set before calling `Render()` |

## Key Functions / Methods

### Render
- Signature: `void Render(overhead_map_data& Control)`
- Purpose: Main entry point; orchestrates rendering of the entire overhead map by calling stage begin/end pairs and element-specific draw methods
- Inputs: `Control` ΓÇö overhead map viewport, scale, origin, dimensions, display mode
- Outputs/Return: None
- Side effects: Calls virtual methods that modify rendering state; may alter automap visibility data
- Calls: Begin/end stage methods, draw methods for polygons/lines/things/player/annotations/map-name, path drawing
- Notes: Implementation in separate .cpp file; handles coordinate transformation and calls configuration-driven drawing

### begin_overall / end_overall (virtual)
- Signature: `virtual void begin_overall()`, `virtual void end_overall()`
- Purpose: Frame initialization and cleanup for the entire map render
- Outputs/Return: None
- Side effects: May set up or tear down graphics state (scissor, viewport, etc.)
- Notes: Empty stubs in base class; subclasses override to manage graphics context

### begin_polygons / end_polygons, begin_lines / end_lines, begin_things / end_things (virtual)
- Signature: `virtual void begin_polygons()`, `virtual void end_polygons()`, etc.
- Purpose: Stage delimiters for batching similar render operations
- Outputs/Return: None
- Side effects: May enable/disable depth testing, set blend modes, etc.
- Notes: Empty stubs; allow backends to optimize state changes

### draw_polygon (virtual)
- Signature: `virtual void draw_polygon(short vertex_count, short *vertices, rgb_color& color)`
- Purpose: Render a single polygon with specified vertices and color
- Inputs: `vertex_count` ΓÇö number of vertices; `vertices` ΓÇö array of endpoint indices; `color` ΓÇö fill color
- Outputs/Return: None
- Side effects: Modifies rendering state (may change texture/color bindings)
- Calls: None (graphics API specific)
- Notes: Receives endpoint indices; use `GetVertex()` to resolve coordinates

### draw_line (virtual)
- Signature: `virtual void draw_line(short *vertices, rgb_color& color, short pen_size)`
- Purpose: Render a line segment with specified color and width
- Inputs: `vertices` ΓÇö array of two endpoint indices; `color` ΓÇö line color; `pen_size` ΓÇö width in pixels
- Outputs/Return: None
- Side effects: Graphics API specific
- Calls: None (backend specific)
- Notes: Used for terrain lines, elevation changes, control panel borders

### draw_thing (virtual)
- Signature: `virtual void draw_thing(world_point2d& center, rgb_color& color, short shape, short radius)`
- Purpose: Render a circular or rectangular entity (monster, item, projectile, checkpoint)
- Inputs: `center` ΓÇö world coordinate; `color` ΓÇö draw color; `shape` ΓÇö `_circle_thing` or `_rectangle_thing`; `radius` ΓÇö extent
- Outputs/Return: None
- Side effects: Graphics API specific
- Calls: None
- Notes: Shape field determines visual representation (circle vs square)

### draw_player (virtual)
- Signature: `virtual void draw_player(world_point2d& center, angle facing, rgb_color& color, short shrink, short front, short rear, short rear_theta)`
- Purpose: Render player avatar on the overhead map with direction indicator
- Inputs: `center` ΓÇö player position; `facing` ΓÇö player heading angle; `color` ΓÇö avatar color; `shrink` ΓÇö size scaling (inverted: larger = more shrunk); `front`, `rear`, `rear_theta` ΓÇö shape parameters
- Outputs/Return: None
- Side effects: Graphics API specific
- Calls: None
- Notes: Called once per frame; `shrink` is `OVERHEAD_MAP_MAXIMUM_SCALE - scale`

### draw_text (virtual)
- Signature: `virtual void draw_text(world_point2d& location, rgb_color& color, char *text, FontSpecifier& FontData, short justify)`
- Purpose: Render text annotation or map title
- Inputs: `location` ΓÇö anchor point; `color` ΓÇö text color; `text` ΓÇö C string; `FontData` ΓÇö font specification; `justify` ΓÇö `_justify_left` or `_justify_center`
- Outputs/Return: None
- Side effects: Graphics API specific
- Calls: None
- Notes: Font and rendering are backend-specific

### set_path_drawing (virtual)
- Signature: `virtual void set_path_drawing(rgb_color& color)`
- Purpose: Prepare for rendering a path polyline
- Inputs: `color` ΓÇö path color
- Outputs/Return: None
- Side effects: May enable line drawing mode
- Calls: None
- Notes: Called before a sequence of `draw_path()` calls

### draw_path (virtual)
- Signature: `virtual void draw_path(short step, world_point2d& location)`
- Purpose: Render a waypoint in a path polyline
- Inputs: `step` ΓÇö segment index (0 for first); `location` ΓÇö waypoint coordinate
- Outputs/Return: None
- Side effects: Graphics API specific
- Calls: None
- Notes: Called multiple times between `set_path_drawing()` and `finish_path()`

### finish_path (virtual)
- Signature: `virtual void finish_path()`
- Purpose: Finalize path rendering
- Outputs/Return: None
- Side effects: May flush buffered geometry
- Calls: None

### GetVertex (static)
- Signature: `static world_point2d& GetVertex(short index)`
- Purpose: Retrieve transformed endpoint coordinate for rendering
- Inputs: `index` ΓÇö endpoint index
- Outputs/Return: Reference to `endpoint_data::transformed` field
- Calls: `get_endpoint_data()`
- Notes: Used internally to resolve vertex coordinates; endpoints must be pre-transformed

### Private inline draw wrappers
Methods like `draw_polygon(short vertex_count, short *vertices, short color, short scale)` are inline overloads that validate color/scale indices, look up configuration data, and delegate to virtual methods with resolved parameters.

**Notes (trivial helpers):**
- `draw_annotation()` ΓÇö Wraps annotation rendering with font and color lookup
- `draw_map_name()` ΓÇö Wraps map title rendering with centered justification
- `set_path_drawing()` ΓÇö Zero-arg overload that uses `ConfigPtr->path_color`

## Control Flow Notes
This is an abstract renderer for the overhead/automap display. It is called during UI rendering, typically once per frame when the map is visible. The `Render()` method (implemented in the corresponding `.cpp` file) orchestrates a multi-stage render:

1. **Overall begin** ΓÇö Graphics context setup
2. **Polygons** ΓÇö Render all visible polygons (colored by type: water, lava, platform, etc.)
3. **Lines** ΓÇö Render edges and elevation changes
4. **Things** ΓÇö Render items, monsters, projectiles, checkpoints
5. **Player** ΓÇö Render player avatar with facing indicator
6. **Annotations** ΓÇö Render text labels
7. **Paths** ΓÇö Render waypoint trails (if enabled)
8. **Overall end** ΓÇö Graphics context cleanup

The "false automap" feature generates a checkpoint-based map: `transform_endpoints_for_overhead_map()` and `generate_false_automap()` support recomputing visibility for checkpoint modes, while `replace_real_automap()` restores the original state. This allows the same renderer to support multiple map discovery modes.

## External Dependencies
- **world.h**: `world_point2d`, `angle` type definitions and vector types
- **map.h**: `endpoint_data`, `get_endpoint_data()`, map geometry accessors
- **monsters.h**: `NUMBER_OF_MONSTER_TYPES` constant
- **overhead_map.h**: `overhead_map_data` structure, scale constants
- **shape_descriptors.h**: `shape_descriptor` type
- **FontHandler.h**: `FontSpecifier` class for font management
- **shell.h**: `_get_player_color()` function and RGBColor type
- **cseries.h**: Standard types, macros, RGB color definitions
