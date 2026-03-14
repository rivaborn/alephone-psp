# Source_Files/RenderOther/motion_sensor.cpp

## File Purpose
Implements the motion sensor displayΓÇöa circular HUD radar that tracks nearby monsters and players. Renders entity "blips" with intensity trails, manages entity lifecycle (appear/fade/disappear), and supports both software and OpenGL rendering backends with customizable monster type classifications.

## Core Responsibilities
- Initialize and reset motion sensor display state per level/player
- Scan world for monsters/players within sensor range using distance and visibility checks
- Track entity positions across multiple frames, managing fade-out animations for departing entities
- Render entity blips using bitmap operations with circular clipping region
- Handle network compass display (multiplayer position indicators)
- Provide XML-based customization of monster type display classes and sensor parameters
- Support both software (pixel-based) and OpenGL rendering implementations

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `motion_sensor_definition` | struct | Settings: update/rescan frequency (ticks), range (world units), scale factor |
| `entity_data` | struct | Tracked entity state: monster index, shape, 6-frame position history, visibility flags, removal delay |
| `region_data` | struct | Circular clipping region: x-clip bounds per y-scanline for circular sensor mask |
| `MonsterDisplays` | static array | Monster type ΓåÆ display class mapping (Friend/Alien/Enemy) |
| `XML_MotSensAssignParser` | class | XML element parser for `<assign monster="X" type="Y"/>` tags |
| `XML_MotSensParser` | class | XML element parser for motion sensor settings (`scale`, `range`, frequencies) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `motion_sensor_settings` | motion_sensor_definition | static | Active sensor config (range, scale, update frequency) |
| `entities` | entity_data* | static | Array of up to 12 tracked entities |
| `sensor_region` | region_data* | static | Pre-computed circular clipping bounds per y-line |
| `motion_sensor_player_index` | short | static | Player index owning this sensor |
| `motion_sensor_side_length` | short | static | Width/height of square sensor display |
| `network_compass_state` | short | static | Current NW/NE/SE/SW quadrant state for multiplayer |
| `mount_shape`, `alien_shapes`, etc. | shape_descriptor | static | Shape IDs for rendering blips and background |
| `ticks_since_last_update`, `ticks_since_last_rescan` | long | static | Timing for update and scan intervals |
| `motion_sensor_changed` | bool | static | Dirty flag for software renderer |

## Key Functions / Methods

### initialize_motion_sensor
- **Signature:** `void initialize_motion_sensor(shape_descriptor mount, shape_descriptor virgin_mounts, shape_descriptor aliens, shape_descriptor friends, shape_descriptor enemies, shape_descriptor compasses, short side_length)`
- **Purpose:** Set up motion sensor display for a level; allocate entity and region arrays, store shape descriptors, pre-compute clipping region.
- **Inputs:** Shape descriptors for mount/blips/compass; display side length (typically ~98 pixels).
- **Outputs/Return:** None; initializes global arrays.
- **Side effects:** Allocates `entities[MAXIMUM_MOTION_SENSOR_ENTITIES]` and `sensor_region[side_length]` arrays; calls `precalculate_sensor_region()`.
- **Calls:** `precalculate_sensor_region()`; uses `new` operator.
- **Notes:** Must be called before `reset_motion_sensor()`. Shapes must be loaded beforehand.

### reset_motion_sensor
- **Signature:** `void reset_motion_sensor(short player_index)`
- **Purpose:** Initialize motion sensor for a specific player; clear display and entity list for level start.
- **Inputs:** Player index whose monitor controls the sensor.
- **Outputs/Return:** None.
- **Side effects:** Copies virgin_mount bitmap to mount; clears entity slots; resets timing counters (`ticks_since_last_*`); resets network compass state.
- **Calls:** `get_shape_bitmap_and_shading_table()` (├ù2); `bitmap_window_copy()`; `MARK_SLOT_AS_FREE()` macro.
- **Notes:** Fails silently if bitmap pointers are NULL; assumes shapes have been loaded.

### HUD_Class::motion_sensor_scan
- **Signature:** `void HUD_Class::motion_sensor_scan(short ticks_elapsed)`
- **Purpose:** Main update loop; rescan world for entities within range, update timing, and invoke render.
- **Inputs:** `ticks_elapsed` = ticks since last call (or `NONE` to force immediate rescan).
- **Outputs/Return:** None.
- **Side effects:** Decrements `ticks_since_last_rescan`; if Γëñ0, iterates all monsters, finds those in range, calls `find_or_add_motion_sensor_entity()`; calls `render_motion_sensor()` for drawing.
- **Calls:** `get_object_data()`, `get_player_data()`, `guess_distance2d()`, `find_or_add_motion_sensor_entity()`, `render_motion_sensor()`.
- **Notes:** Uses `MONSTER_IS_PLAYER()` and `MONSTER_IS_ACTIVE()` macros; only adds visible entities; always calls render regardless of rescan interval.

### HUD_SW_Class::render_motion_sensor / HUD_OGL_Class::render_motion_sensor
- **Signature:** 
  - `void HUD_SW_Class::render_motion_sensor(short ticks_elapsed)` 
  - `void HUD_OGL_Class::render_motion_sensor(short ticks_elapsed)`
- **Purpose:** Render motion sensor display (software vs. OpenGL implementation).
- **Inputs:** `ticks_elapsed` for timing/update interval.
- **Outputs/Return:** None.
- **Side effects (SW):** Decrements update timer; if expired, erases old blips, draws compass, draws new blips, sets `motion_sensor_changed = true`. (OGL):** Always draws background and entities each frame; sets `motion_sensor_changed` only if update interval expired.
- **Calls:** `erase_all_entity_blips()`, `draw_network_compass()`, `draw_all_entity_blips()`, `DrawShapeAtXY()` (OGL only).
- **Notes:** Software version uses dirty flag for efficiency; OGL always redraws. Timer resets to `MOTION_SENSOR_UPDATE_FREQUENCY`.

### erase_all_entity_blips
- **Signature:** `void HUD_Class::erase_all_entity_blips(void)`
- **Purpose:** Update entity positions and visibility, erase outdated blips, mark entities for removal if out of range.
- **Inputs:** None; reads global entity list and player data.
- **Outputs/Return:** None.
- **Side effects:** 
  - Shifts `visible_flags` and `previous_points` arrays by 1 for trail history.
  - Erases blip at `visible_flags[NUMBER_OF_PREVIOUS_LOCATIONS-1]` if set.
  - Checks if entity is still in range; marks as "being removed" if not.
  - Calculates 2D sensor position from 3D world position (transforms relative to player, scales).
  - Tracks visibility based on invisibility transfer mode and magnetic flickering.
  - Increments `remove_delay` for removing entities; frees slot when delay expires.
- **Calls:** `erase_entity_blip()`, `transform_point2d()`, `SLOT_IS_BEING_REMOVED()` macro.
- **Notes:** Maintains fade-out effect; uses `FLICKER_FREQUENCY` (0xf) for magnetic environment.

### draw_all_entity_blips
- **Signature:** `void HUD_Class::draw_all_entity_blips(void)`
- **Purpose:** Render all tracked entities with intensity-based trails.
- **Inputs:** None; reads global entity list.
- **Outputs/Return:** None.
- **Side effects:** Iterates entities and visibility history in reverse intensity order (5ΓåÆ0), drawing each visible blip.
- **Calls:** `draw_entity_blip()`.
- **Notes:** Higher intensities (older history) drawn first, allowing newer blips to overwrite; gives depth effect.

### draw_entity_blip (SW/OGL variants) / erase_entity_blip (SW only)
- **Signature:** 
  - `void HUD_SW_Class::draw_entity_blip(point2d *location, shape_descriptor shape)`
  - `void HUD_OGL_Class::draw_entity_blip(point2d *location, shape_descriptor shape)`
  - `void HUD_SW_Class::erase_entity_blip(point2d *location, shape_descriptor shape)`
- **Purpose:** Draw or erase a single entity blip on the motion sensor.
- **Inputs:** 2D location on sensor (center coordinates), shape descriptor for blip graphic.
- **Outputs/Return:** None.
- **Side effects (SW draw):** Calls `clipped_transparent_sprite_copy()` to blend blip onto mount with circular clipping. (SW erase):** Calls `bitmap_window_copy()` to restore virgin background. (OGL):** Calls `DrawShapeAtXY()` with clipping plane setup.
- **Calls:** Various bitmap/sprite copy functions; `SetClipPlane()` / `DisableClipPlane()` (OGL).
- **Notes:** SW renderer erases by restoring background; OGL draws directly. Blips are centered at location.

### find_or_add_motion_sensor_entity
- **Signature:** `static short find_or_add_motion_sensor_entity(short monster_index)`
- **Purpose:** Track a new or existing entity; allocate entity slot and initialize state.
- **Inputs:** Monster index to track.
- **Outputs/Return:** Entity array index (or `NONE` if no free slots).
- **Side effects:** If not already tracked, finds first free entity slot, initializes flags, shape, location, and visibility arrays.
- **Calls:** `get_motion_sensor_entity_shape()`, `get_monster_data()`, `get_object_data()`.
- **Notes:** Ignores entities being removed; returns `NONE` if no free slots (max 12).

### get_motion_sensor_entity_shape
- **Signature:** `static shape_descriptor get_motion_sensor_entity_shape(short monster_index)`
- **Purpose:** Determine which blip shape to display for an entity (friend/alien/enemy).
- **Inputs:** Monster index.
- **Outputs/Return:** Shape descriptor for the appropriate blip graphic.
- **Side effects:** None.
- **Calls:** `MONSTER_IS_PLAYER()` macro; `get_monster_data()`, `get_player_data()`, `monster_index_to_player_index()`, `GET_GAME_OPTIONS()`, `GET_GAME_TYPE()` macros.
- **Notes:** Players show as friendly if same team or cooperative; otherwise enemy. Non-players use `MonsterDisplays` lookup table.

### precalculate_sensor_region
- **Signature:** `static void precalculate_sensor_region(short side_length)`
- **Purpose:** Pre-compute circular clipping bounds for efficient sprite rendering.
- **Inputs:** Sensor display side length (width/height in pixels).
- **Outputs/Return:** None; fills `sensor_region[]` array.
- **Side effects:** For each y-line [0, side_length), calculates x-clip bounds as `[center - radius, center + radius]` for circle equation.
- **Calls:** `sqrt()` from math.h.
- **Notes:** Uses floating-point math for accuracy; stores as int16. Radius is `(side_length/2) + 1`.

### Bitmap copy helpers (bitmap_window_copy, clipped_transparent_sprite_copy, unclipped_solid_sprite_copy)
- **Purpose:** Low-level pixel blitting for rendering and erasing blips.
- **Inputs:** Source/dest bitmaps, coordinates, optional clipping region.
- **Outputs/Return:** None.
- **Side effects:** Direct pixel-level manipulation; assumes row_addresses are valid.
- **Notes:** Bitmap copy functions assume source and dest have same dimensions; clipped variants check bounds; transparent sprite copy skips pixels with index 0.

### XML parser classes (XML_MotSensAssignParser, XML_MotSensParser)
- **Purpose:** Parse `<motion_sensor>` XML element and child `<assign>` tags for customization.
- **Key methods:** `Start()` (backup original values), `HandleAttribute()` (parse XML attributes), `AttributesDone()` (validate and apply), `ResetValues()` (restore backup).
- **Side effects:** Modify global `MonsterDisplays[]` and `motion_sensor_settings` based on XML.
- **Notes:** Supports per-monster-type shape assignment and sensor parameter tuning (scale, range, frequency).

## Control Flow Notes
- **Initialization:** `initialize_motion_sensor()` ΓåÆ `reset_motion_sensor()` when level loads.
- **Per-frame update (in game loop):** `motion_sensor_scan()` is called each tick, which handles rescan timing and delegates rendering.
- **Rendering:** Separate code paths for software (`HUD_SW_Class`) and OpenGL (`HUD_OGL_Class`); software uses dirty flag for optimization, OpenGL redraws every frame.
- **Entity lifecycle:** New entities added on rescan ΓåÆ positions tracked each frame ΓåÆ fade-out over 6 frames when departing ΓåÆ slot reclaimed.
- **Trail effect:** 6-element history per entity; drawn in reverse order (oldestΓåÆnewest) to create intensity gradient.

## External Dependencies
- **Notable includes:** `monsters.h` (monster/player data), `map.h` (world geometry), `render.h` (view/rendering), `interface.h` (shapes, bitmaps), `player.h` (player data), `network_games.h` (compass state), `HUDRenderer_SW.h` / `HUDRenderer_OGL.h` (renderer-specific drawing).
- **Key external functions/symbols:** `get_object_data()`, `get_monster_data()`, `get_player_data()`, `guess_distance2d()`, `transform_point2d()`, `get_shape_bitmap_and_shading_table()`, `DrawShapeAtXY()` (OGL), `get_interface_rectangle()`, `get_network_compass_state()` (defined elsewhere).
- **Game world globals:** `monsters[]`, `dynamic_world->tick_count`, `static_world->environment_flags` (magnetic environment check).
