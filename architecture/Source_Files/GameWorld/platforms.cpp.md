# Source_Files/GameWorld/platforms.cpp

## File Purpose
Implements platform (moving surface/door) simulation for the game world. Handles creation, activation, movement with collision detection, state management, and interaction with players and monsters. Includes serialization and XML configuration support.

## Core Responsibilities
- **Platform lifecycle**: creation (`new_platform`), initialization, state management
- **Physics simulation**: height changes, collision detection with obstruction handling, directional reversal
- **Interaction layer**: player/monster activation, entry/exit validation with pathfinding heuristics
- **Cascading control**: adjacent platform activation/deactivation based on static flags
- **Geometry updates**: endpoint/line heights, texture adjustments, media height tracking
- **Audio**: platform-specific sound effects (starting, stopping, obstructed, uncontrollable)
- **Serialization**: binary pack/unpack and XML configuration parsing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `platform_data` | struct | Runtime state: type, heights, flags, position, parent/child links, endpoint owners |
| `static_platform_data` | struct | Design-time config: type, speed, delay, height limits, flags, tag |
| `platform_definition` | struct | Type defaults: sounds, damage, key requirements (from `platform_definitions.h`) |
| `endpoint_owner_data` | struct | Cached polygon/line adjacency for a platform vertex |
| `XML_PlatformParser` | class | XML element parser for platform type customization |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platforms` | `vector<platform_data>` (macro) | global | All active platforms indexed by `platform_index` |
| `platform_definitions` | array/vector | global | Static definitions for each platform type (speed, sounds, damage) |
| `original_platform_definitions` | pointer | static | Backup of original definitions before XML modifications |
| `PlatformParser` | `XML_PlatformParser` | static | Singleton parser for XML platform elements |

## Key Functions / Methods

### new_platform
- **Signature**: `short new_platform(struct static_platform_data *data, short polygon_index)`
- **Purpose**: Create and initialize a platform from static data in a given polygon
- **Inputs**: `data` (type, speed, delay, flags, tag), `polygon_index` (target polygon)
- **Outputs/Return**: platform index if created; `NONE` if platform count exceeds limit
- **Side effects**: Increments `dynamic_world->platform_count`, modifies polygon (type, permutation), calculates initial heights, adjusts endpoint/line heights
- **Calls**: `get_polygon_data`, `calculate_platform_extrema`, `calculate_endpoint_polygon_owners`, `calculate_endpoint_line_owners`, `adjust_platform_endpoint_and_line_heights`, `adjust_platform_for_media`
- **Notes**: Handles both floor-rising and ceiling-lowering platforms; can be a "do-nothing" platform

### update_platforms
- **Signature**: `void update_platforms(void)`
- **Purpose**: Main per-tick update loop for all active platforms
- **Inputs**: None (reads from global `platforms`, `dynamic_world`)
- **Outputs/Return**: None (modifies platform state, plays sounds, triggers adjacent activations)
- **Side effects**: Modifies `floor_height`, `ceiling_height`, dynamic flags; calls `change_polygon_height`, `adjust_platform_sides`, `adjust_platform_endpoint_and_line_heights`, `set_adjacent_platform_states`, `set_platform_state`
- **Calls**: `get_polygon_data`, `get_platform_definition`, `change_polygon_height`, `adjust_platform_sides`, `adjust_platform_endpoint_and_line_heights`, `adjust_platform_for_media`, `take_out_the_garbage`, `play_platform_sound`
- **Notes**: Handles direction reversal on obstruction, deactivation at extremes, media interaction; processes delta_height based on extend/contract direction and speed

### set_platform_state
- **Signature**: `bool set_platform_state(short platform_index, bool state, short parent_platform_index)`
- **Purpose**: Activate or deactivate a platform (internal state setter)
- **Inputs**: `platform_index` (target), `state` (true=activate, false=deactivate), `parent_platform_index` (origin of cascading activation)
- **Outputs/Return**: New state after change (bool)
- **Side effects**: Sets `_platform_is_active` flag, triggers light activation/deactivation, cascades to adjacent platforms, calls Lua hook `L_Call_Platform_Activated`, updates switch positions
- **Calls**: `get_platform_data`, `set_light_status`, `set_adjacent_platform_states`, `assume_correct_switch_position`, `play_platform_sound`
- **Notes**: Prevents re-triggering in same tick via `_platform_was_just_activated_or_deactivated` flag; respects `_platform_activates_only_once` and parent-blocking flags

### player_touch_platform_state
- **Signature**: `void player_touch_platform_state(short player_index, short platform_index)`
- **Purpose**: Handle player action key activation of a platform
- **Inputs**: `player_index`, `platform_index`
- **Outputs/Return**: None (may change platform state or play sound)
- **Side effects**: May activate/deactivate, reverse direction, consume key item; plays appropriate sound
- **Calls**: `get_platform_data`, `get_platform_definition`, `try_and_subtract_player_item`, `set_platform_state`, `play_platform_sound`
- **Notes**: Respects `_platform_is_player_controllable` and `_platform_cannot_be_externally_deactivated`; checks for key requirement

### monster_can_enter_platform / monster_can_leave_platform
- **Signature**: `short monster_can_enter_platform(platform_index, source_polygon_index, height, min_ledge, max_ledge)` and symmetric leave function
- **Purpose**: AI pathfinding: determine if monster can traverse to/from a platform
- **Inputs**: `platform_index`, adjacent polygon indices, entity height, ledge height constraints
- **Outputs/Return**: Accessibility code (will_never, will_be, is, etc.)
- **Side effects**: None (read-only)
- **Calls**: `get_polygon_data`, `get_platform_data`
- **Notes**: Accounts for door vs. non-door behavior; uses min/max heights for contracted/extended states

### adjust_platform_endpoint_and_line_heights
- **Signature**: `void adjust_platform_endpoint_and_line_heights(short platform_index)`
- **Purpose**: Recalculate and cache line/endpoint heights after platform moves
- **Inputs**: `platform_index`
- **Outputs/Return**: None (modifies endpoint/line data)
- **Side effects**: Updates `line->highest_adjacent_floor`, `line->lowest_adjacent_ceiling`, `endpoint->highest_adjacent_floor_height`, `endpoint->lowest_adjacent_ceiling_height`; sets line transparency/solidity
- **Calls**: `get_platform_data`, `get_polygon_data`, `get_endpoint_data`, `get_line_data`, `get_map_indexes`
- **Notes**: Handles variable-elevation lines; caches adjacent polygon heights for rendering and collision

### calculate_platform_extrema
- **Signature**: `void calculate_platform_extrema(platform_index, lowest_level, highest_level)`
- **Purpose**: Compute min/max height bounds for platform movement based on adjacent geometry
- **Inputs**: `platform_index`, user-specified level bounds (or `NONE` to auto-calculate)
- **Outputs/Return**: None (modifies `platform->minimum_floor_height`, etc.)
- **Side effects**: Stores calculated min/max floor/ceiling heights
- **Calls**: `get_platform_data`, `get_polygon_data`
- **Notes**: Handles split platforms (floor+ceiling), floor-only, ceiling-only, and native polygon height modes

### adjust_platform_for_media
- **Signature**: `void adjust_platform_for_media(short platform_index, bool initialize)`
- **Purpose**: Track platform height relative to liquid/media surface
- **Inputs**: `platform_index`, `initialize` (true on creation, false on move)
- **Outputs/Return**: None (sets platform flags and plays sound)
- **Side effects**: Sets `_platform_floor_below_media` and `_platform_ceiling_below_media`; plays media enter/leave sounds
- **Calls**: `get_platform_data`, `get_polygon_data`, `get_media_data`, `play_polygon_sound`, `get_media_sound`
- **Notes**: Only processes if polygon has media; media height is dynamic per light intensity

### set_adjacent_platform_states
- **Signature**: `void set_adjacent_platform_states(platform_index, state)`
- **Purpose**: Recursively activate/deactivate adjacent platforms based on configuration
- **Inputs**: `platform_index`, target state (bool)
- **Outputs/Return**: None (cascades state changes)
- **Side effects**: Calls `set_platform_state` for each adjacent platform respecting parent-blocking
- **Calls**: `get_platform_data`, `get_polygon_data`, `polygon_index_to_platform_index`, `set_platform_state`
- **Notes**: Prevents infinite loops via parent-blocking flag

### adjust_platform_sides
- **Signature**: `void adjust_platform_sides(platform_index, old_ceiling_height, new_ceiling_height)`
- **Purpose**: Adjust side texture coordinates (y-offset) as ceiling platform moves
- **Inputs**: `platform_index`, old and new ceiling heights
- **Outputs/Return**: None (modifies side texture data)
- **Side effects**: Updates `side->primary_texture.y0` for platform and shaft sides based on height delta
- **Calls**: `get_platform_data`, `get_polygon_data`, `get_line_data`, `get_side_data`
- **Notes**: Handles split, high, low, and full side types differently

**Trivial/Helper functions** (summarized):
- `get_platform_data()`, `get_platform_definition()` ΓÇö bounds-checked accessors
- `platform_is_on()` ΓÇö query if platform is active
- `platform_is_legal_player_target()` ΓÇö check if player can activate (respects one-time flag)
- `platform_is_at_initial_state()` ΓÇö check if at resting position
- `try_and_change_platform_state()`, `try_and_change_tagged_platform_states()` ΓÇö wrappers with validation
- `play_platform_sound()` ΓÇö map sound type to definition and dispatch
- `polygon_index_to_platform_index()` ΓÇö lookup via linear search
- `take_out_the_garbage()` ΓÇö remove garbage objects from platform polygon
- `unpack_platform_data()`, `pack_platform_data()` ΓÇö serialization helpers
- XML parser methods (`Start()`, `HandleAttribute()`, `AttributesDone()`, `ResetValues()`)

## Control Flow Notes
Platforms operate in the **update/frame loop** via `update_platforms()` called each game tick. Activation is triggered by:
- **Player action key**: `player_touch_platform_state()` ΓåÆ `try_and_change_platform_state()` ΓåÆ `set_platform_state()`
- **Monster entry**: `platform_was_entered()` ΓåÆ `try_and_change_platform_state()`
- **Polygon type triggers**: external code calls `set_platform_state()`
- **Lua scripting**: `L_Call_Platform_Activated()` hook invoked on state change

Movement is continuous per tick with delta-height applied, obstruction detection via `change_polygon_height()`, and cascading adjacent activations on reach-extremes or explicit flags.

## External Dependencies
- **world.h**: `world_distance`, `world_point3d`, angle types
- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `change_polygon_height()`, accessors
- **platforms.h**: Type definitions, flag macros
- **lightsource.h**: `set_light_status()`, light data
- **SoundManager.h**: Sound playback, `CauseAmbientSoundSourceUpdate()`
- **player.h**: `try_and_subtract_player_item()` for key requirements
- **media.h**: Media data and sound helpers
- **lua_script.h**: `L_Call_Platform_Activated()` hook
- **DamageParser.h**: Damage parsing for obstruction damage
- **XML_ElementParser.h**: XML configuration support

**Defined elsewhere**: `platform_definitions` array (from `platform_definitions.h`), `dynamic_world` global, `platforms` vector, `change_polygon_height()`, light/media/object accessors
