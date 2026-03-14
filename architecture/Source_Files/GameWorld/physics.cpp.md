# Source_Files/GameWorld/physics.cpp

## File Purpose
Implements player physics simulation for the Marathon/Aleph One game engine. Handles movement input processing, collision detection, gravity, jumping, falling, and camera positioning. Encapsulates physics model parameters (velocities, accelerations) and applies them each frame with proper collision response and external force handling.

## Core Responsibilities
- Initialize and update player physics state each tick
- Process action flags (movement, turning, jumping) into velocity/acceleration
- Perform collision detection against walls, objects, and ground
- Calculate and apply gravity, jumping, and climbing forces
- Manage player on-ground vs. airborne states
- Generate camera position with head bobbing and pitch/yaw
- Handle external forces (damage knockback, momentum)
- Pack/unpack physics constants for serialization

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| physics_variables | struct | Player motion state (position, velocity, direction, rotation angles, external forces) |
| physics_constants | struct | Physics parameters (max velocities, accelerations, gravity, terminal velocity) |
| player_data | struct (external) | Main player record; contains physics_variables and object references |
| object_data | struct (external) | World object with position, facing, polygon residence |
| polygon_data | struct (external) | Map polygon with floor/ceiling heights, media index |
| fixed_point3d | typedef (external) | Fixed-point 3D vector |
| fixed_vector3d | struct (external) | 3D velocity with i, j, k components |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| saved_points | world_point3d* | static (DEBUG only) | Saved player positions for divergence checking |
| saved_thetas | angle* | static (DEBUG only) | Saved player facing angles for divergence checking |
| saved_point_count | short | static (DEBUG only) | Count of saved points in current iteration |
| saved_point_iterations | short | static (DEBUG only) | Number of times physics has been initialized |
| saved_divergence_warning | bool | static (DEBUG only) | Flag to suppress repeated divergence warnings |
| physics_models | physics_constants[] | extern (defined in physics_models.h) | Array of physics parameter sets |

## Key Functions / Methods

### initialize_player_physics_variables
- Signature: `void initialize_player_physics_variables(short player_index)`
- Purpose: One-time initialization of physics state when player enters map.
- Inputs: `player_index` ΓÇô which player to initialize
- Outputs/Return: None; modifies playerΓåÆvariables and objectΓåÆlocation in place
- Side effects: Zeroes step phase/amplitude, sets position from object location, clears flags, allocates DEBUG divergence arrays
- Calls: `get_player_data()`, `get_monster_data()`, `get_object_data()`, `get_physics_constants_for_model()`, `instantiate_physics_variables()`
- Notes: Invoked once per level/respawn. Sets up shadow variables and collision state.

### update_player_physics_variables
- Signature: `void update_player_physics_variables(short player_index, uint32 action_flags, bool predictive)`
- Purpose: Main entry point to advance player physics by one tick; applies movement and updates world state.
- Inputs: `player_index`, `action_flags` (movement/turn/jump bits), `predictive` (if true, minimize state changes for prediction rollback)
- Outputs/Return: None; updates player position, velocity, and object world state
- Side effects: Modifies player variables, translates object in world, updates polygon residence, records divergence (DEBUG)
- Calls: `physics_update()`, `instantiate_physics_variables()`
- Notes: Called once per frame. If `predictive=true`, skips side effects like `changed_polygon()` and `monster_moved()`.

### physics_update
- Signature: `static void physics_update(struct physics_constants *constants, struct physics_variables *variables, struct player_data *player, uint32 action_flags)`
- Purpose: Core physics loop; applies forces, velocities, and constraints without touching world geometry.
- Inputs: `constants` (physics params for run/walk mode), `variables` (player state), `player` (for dead-player checks), `action_flags` (input)
- Outputs/Return: None; modifies `variables` (position, velocity, elevation, step phase, flags)
- Side effects: Updates position, velocity, direction, step phase; manages recentering and airborne state
- Calls: Trigonometric lookups (cosine_table, sine_table); `isqrt()` for drag calculation
- Notes: Runs locally on fixed-point vectors; does not interact with map. Handles dead player special case (reduced upward look speed). Contains complex velocity clamping logic and auto-recentering with user preference check.

### instantiate_physics_variables
- Signature: `static void instantiate_physics_variables(struct physics_constants *constants, struct physics_variables *variables, short player_index, bool first_time, bool take_action)`
- Purpose: Apply calculated physics to world: collision detection, polygon crossing, camera positioning.
- Inputs: `constants`, `variables`, `player_index`, `first_time` (skip polygon-change triggers), `take_action` (enable side effects)
- Outputs/Return: None; updates object position and polygon, sets playerΓåÆcamera_location, updates media flags
- Side effects: Calls `keep_line_segment_out_of_walls()`, `legal_player_move()`, `translate_map_object()`, `changed_polygon()`, `monster_moved()`, `bump_monster()`; updates floor/ceiling/media heights in variables
- Calls: Map collision and movement functions; audio positioning
- Notes: Where physics meets the world. Performs 2D wall collision clipping, object collision, media detection. Builds camera location with step-bob applied (zero if chase-cam active).

### get_physics_constants_for_model
- Signature: `static struct physics_constants *get_physics_constants_for_model(short physics_model, uint32 action_flags)`
- Purpose: Retrieve physics parameters for current model and movement state (running vs. walking).
- Inputs: `physics_model` (editor or earth gravity), `action_flags` (to detect run bit)
- Outputs/Return: Pointer into global `physics_models` array
- Side effects: None
- Calls: None
- Notes: Simple switch statement; asserts on invalid model.

### mask_in_absolute_positioning_information
- Signature: `uint32 mask_in_absolute_positioning_information(uint32 action_flags, _fixed delta_yaw, _fixed delta_pitch, _fixed delta_position)`
- Purpose: Encode relative position/rotation deltas into action flags for network synchronization.
- Inputs: `action_flags`, three delta values in [ΓêÆFIXED_ONE, FIXED_ONE]
- Outputs/Return: Modified action_flags with absolute positioning bits set and mode flags enabled
- Side effects: None
- Calls: None
- Notes: Quantizes deltas to fixed bit widths (ABSOLUTE_YAW_BITS, etc.); clamps to prevent drift; used for continuous lag-tolerant netplay.

### accelerate_player
- Signature: `void accelerate_player(short monster_index, world_distance vertical_velocity, angle direction, world_distance velocity)`
- Purpose: Apply external force (e.g., knockback, wind) to player.
- Inputs: `monster_index` (to find player), vertical and horizontal velocity to add
- Outputs/Return: None; updates playerΓåÆvariablesΓåÆexternal_velocity
- Side effects: Clamps vertical velocity to terminal velocity
- Calls: Trigonometric lookups for direction decomposition
- Notes: Used for impacts, explosions, and environmental forces.

### get_binocular_vision_origins
- Signature: `void get_binocular_vision_origins(short player_index, world_point3d *left, short *left_polygon_index, angle *left_angle, world_point3d *right, short *right_polygon_index, angle *right_angle)`
- Purpose: Calculate left and right eye camera positions for stereoscopic rendering.
- Inputs: `player_index`
- Outputs/Return: Six output parameters filled with eye position, polygon, and facing angle
- Side effects: Calls `find_new_object_polygon()` to locate eyes in world
- Calls: Trigonometric lookups; polygon collision detection
- Notes: Eyes are offset perpendicular to player facing by `half_camera_separation` from physics_constants.

### get_absolute_pitch_range
- Signature: `void get_absolute_pitch_range(_fixed *minimum, _fixed *maximum)`
- Purpose: Return the vertical look angle limits.
- Inputs: Output pointers
- Outputs/Return: Sets min/max to ┬▒maximum_elevation from physics constants
- Side effects: None
- Calls: `get_physics_constants_for_model()`

### get_player_forward_velocity_scale
- Signature: `_fixed get_player_forward_velocity_scale(short player_index)`
- Purpose: Return player's forward velocity as fraction of maximum (for HUD/animation).
- Inputs: `player_index`
- Outputs/Return: _fixed in [ΓêÆFIXED_ONE, FIXED_ONE]
- Side effects: None
- Calls: Trigonometric lookups for direction projection
- Notes: Projects velocity vector onto facing direction.

### adjust_player_for_polygon_height_change
- Signature: `void adjust_player_for_polygon_height_change(short monster_index, short polygon_index, world_distance new_floor_height, world_distance new_ceiling_height)`
- Purpose: Adjust player height when supporting polygon's floor changes (platform elevation).
- Inputs: Monster/polygon indices, new heights
- Outputs/Return: None; updates playerΓåÆvariablesΓåÆposition.z and external_velocity if dead
- Side effects: Teleports player vertically to maintain stance on moving floor
- Calls: None (simple arithmetic)
- Notes: Only affects player if in specified polygon.

## Serialization Functions (unpack/pack)

### unpack_physics_constants, pack_physics_constants
- Purpose: Deserialize/serialize physics_constants structures from/to byte streams.
- Inputs: Byte pointer, Count, optional Objects array (defaults to global physics_models)
- Outputs/Return: Advanced byte pointer
- Side effects: Reads/writes 26 fields per structure in fixed order; asserts correct byte count
- Notes: Uses StreamToValue/ValueToStream macros for endian-safe I/O.

## Control Flow Notes
- **Per-level init**: `initialize_player_physics_variables()` called once when player spawns.
- **Per-tick update**: `update_player_physics_variables()` called by `update_players()` in main game loop.
  - Calls `physics_update()` to calculate new velocities/position in local space.
  - Calls `instantiate_physics_variables()` to apply collisions and snap to world.
- **Collision/world sync**: Polygon changes, media detection, and object interactions happen in `instantiate_physics_variables()`.
- **Camera setup**: Done at end of `instantiate_physics_variables()`; ready for render.

## External Dependencies
- **Includes**: `cseries.h` (common types, macros), `render.h`, `map.h`, `player.h`, `interface.h`, `monsters.h`, `media.h`, `ChaseCam.h`, `Packing.h`, `<string.h>`
- **External symbols used**:
  - `get_player_data()`, `get_monster_data()`, `get_object_data()` ΓÇô data accessors
  - `get_polygon_data()` ΓÇô map geometry
  - `get_media_data()` ΓÇô media (liquid) properties
  - `keep_line_segment_out_of_walls()`, `legal_player_move()`, `translate_map_object()`, `find_new_object_polygon()` ΓÇô collision/movement (defined elsewhere in map.c)
  - `changed_polygon()`, `monster_moved()`, `bump_monster()` ΓÇô game logic callbacks (defined elsewhere)
  - `cosine_table[]`, `sine_table[]` ΓÇô trigonometric lookups (defined elsewhere)
  - `isqrt()` ΓÇô integer square root (defined elsewhere)
  - `static_world`, `dynamic_world` ΓÇô global game state
  - `local_player` ΓÇô shortcut to local player data
  - `ChaseCam_IsActive()` ΓÇô camera mode check
  - `physics_models`, `original_physics_models` ΓÇô physics parameter tables (imported from physics_models.h)
