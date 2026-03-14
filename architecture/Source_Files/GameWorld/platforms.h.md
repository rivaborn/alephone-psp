# Source_Files/GameWorld/platforms.h

## File Purpose
Defines platform (moving doors and platforms) structures, enumerations, and behavior flags for the game world. Declares all platform-related functions and provides serialization utilities. Platforms are dynamic geometry pieces that can extend/contract, activate/deactivate, and interact with players and monsters.

## Core Responsibilities
- Define platform type enumeration (spht doors, pfhor doors, platforms, etc.)
- Define speed/delay preset enumerations for platform motion
- Define static configuration flags (27 bits) for platform properties
- Define dynamic runtime flags (10 bits) for platform state tracking
- Provide flag-testing and flag-setting macros for both static and dynamic flags
- Declare platform lifecycle functions (creation, activation, state changes)
- Declare platform-entity interaction queries (can monster enter/leave, can player use)
- Declare serialization/deserialization functions for save/load
- Declare XML parser accessor for loading platform definitions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `endpoint_owner_data` | struct | Maps platform endpoints to polygon and line indices they affect |
| `static_platform_data` | struct | Immutable platform configuration (32 bytes): type, speed, delay, height bounds, flags, polygon index |
| `platform_data` | struct | Complete platform state (140 bytes): combines static config + runtime state (floor/ceiling heights, dynamic flags, endpoint owners, parent platform) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlatformList` | `vector<platform_data>` | global | All active platforms in the current map |

## Key Functions / Methods

### new_platform
- Signature: `short new_platform(struct static_platform_data *data, short polygon_index)`
- Purpose: Create and initialize a new platform instance in the map
- Inputs: Platform configuration (type, speed, delay, heights, flags), target polygon
- Outputs/Return: Platform index if successful, or error code
- Side effects: Appends to `PlatformList`, allocates endpoint owner data
- Calls: (not visible in this file)

### update_platforms
- Signature: `void update_platforms(void)`
- Purpose: Main per-frame update loop for all platforms; moves active platforms, handles collisions, triggers adjacent platforms
- Inputs: None (reads global `PlatformList`)
- Outputs/Return: None
- Side effects: Updates floor/ceiling heights, dynamic flags, plays sounds, modifies `PlatformList` states
- Calls: (implementation in platforms.c)

### platform_was_entered
- Signature: `void platform_was_entered(short platform_index, bool player)`
- Purpose: Called when an entity enters the platform's polygon; triggers activation logic
- Inputs: Platform index, whether the entity is a player
- Outputs/Return: None
- Side effects: May activate the platform or adjacent platforms
- Calls: (not visible)

### try_and_change_platform_state
- Signature: `bool try_and_change_platform_state(short platform_index, bool state)`
- Purpose: Attempt to activate or deactivate a single platform
- Inputs: Platform index, desired state (true=active, false=inactive)
- Outputs/Return: Whether state change succeeded
- Side effects: Updates dynamic flags, triggers adjacent platforms if configured
- Calls: (not visible)

### try_and_change_tagged_platform_states
- Signature: `bool try_and_change_tagged_platform_states(short tag, bool state)`
- Purpose: Activate or deactivate all platforms sharing a given tag
- Inputs: Tag ID, desired state
- Outputs/Return: Whether change succeeded
- Side effects: (as above, for all matching platforms)
- Calls: (not visible)

### monster_can_enter_platform / monster_can_leave_platform
- Signature: `short monster_can_enter_platform(short platform_index, short source_polygon_index, world_distance height, world_distance minimum_ledge_delta, world_distance maximum_ledge_delta)` (and leave variant)
- Purpose: Query whether a monster can physically enter or leave a platform given current geometry
- Inputs: Platform index, monster's current/target polygon, height, ledge constraints
- Outputs/Return: Enumerated accessibility state (_will_never_be_accessable, _will_be_accessable, _might_be_accessable, _is_accessable, and exit variants)
- Side effects: None (query only)
- Calls: (not visible)

### get_platform_data
- Signature: `platform_data *get_platform_data(short platform_index)`
- Purpose: Retrieve runtime data for a platform by index
- Inputs: Platform index
- Outputs/Return: Pointer to platform_data struct
- Side effects: None
- Calls: (likely simple array access)

### adjust_platform_for_media
- Signature: `void adjust_platform_for_media(short platform_index, bool initialize)`
- Purpose: Recalculate platform heights when water/lava level changes
- Inputs: Platform index, whether this is initialization
- Outputs/Return: None
- Side effects: Updates floor/ceiling heights if platform is partially submerged
- Calls: (not visible)

### platform_is_on
- Signature: `bool platform_is_on(short platform_index)`
- Purpose: Check if platform is currently active and moving
- Inputs: Platform index
- Outputs/Return: Boolean
- Side effects: None (query only)
- Calls: (likely macro-based flag check)

### player_touch_platform_state
- Signature: `void player_touch_platform_state(short player_index, short platform_index)`
- Purpose: Handle player activating/toggling a platform via action key (for player-controllable platforms)
- Inputs: Player index, platform index
- Outputs/Return: None
- Side effects: Toggles platform state if `_platform_is_player_controllable` flag is set
- Calls: (not visible)

### Pack/Unpack functions
- Signature: `uint8 *unpack_static_platform_data(uint8 *Stream, static_platform_data *Objects, size_t Count)` (and variants for pack, platform_data)
- Purpose: Serialize/deserialize platform definitions to/from binary stream (for save files, network)
- Inputs: Data stream, object array, count
- Outputs/Return: Updated stream pointer
- Side effects: Reads from or writes to stream
- Calls: (implementation in platforms.c)

### Platforms_GetParser
- Signature: `XML_ElementParser *Platforms_GetParser()`
- Purpose: Return parser object for loading platforms from XML map definitions
- Inputs: None
- Outputs/Return: Pointer to XML parser instance
- Side effects: None (factory)
- Calls: (not visible)

## Notes
- **Static flags** (27 bits in uint32): Configure immutable properties (initially active, extends floor-to-ceiling, damage on obstruction, lighting, controllability, etc.)
- **Dynamic flags** (10 bits in uint16): Track runtime state (is active, is extending, is moving, fully extended/contracted, below media, etc.)
- Extensive macro suite for flag testing (`PLATFORM_IS_ACTIVE(p)`, `PLATFORM_IS_EXTENDING(p)`, etc.) and setting (`SET_PLATFORM_IS_ACTIVE(p, v)`)
- Macros like `PLATFORM_IS_CONTRACTING(p)` derive inverted state from extension flag
- Compound macros like `CLEAR_PLATFORM_POSITIONING_FLAGS(p)` update multiple flags atomically
- Platform graph: supports parent platform references and adjacent platform activation (when activating/deactivating/at each level)
- Return enum for monster accessibility distinguishes "will never" vs. "might" vs. "currently is" accessible

## Control Flow Notes
Platforms integrate into the main game loop via `update_platforms()` called each frame. Entity-platform interactions trigger via `platform_was_entered()` and `player_touch_platform_state()`. State changes propagate through adjacent platform activation based on static flags. Serialization hooks (`unpack_*`) are invoked during map load.

## External Dependencies
- `XML_ElementParser.h` ΓÇö base class for XML parsing (platform definitions)
- Custom scalar types: `int16`, `uint16`, `uint32`, `world_distance`
- Macros: `TEST_FLAG16()`, `TEST_FLAG32()`, `SET_FLAG16()`, `SET_FLAG32()` ΓÇö defined elsewhere (flag manipulation utilities)
- Constants: `TICKS_PER_SECOND`, `WORLD_ONE` ΓÇö time and spatial units (defined elsewhere)
- Macro `MAXIMUM_VERTICES_PER_POLYGON` ΓÇö polygon geometry limit
- STL `vector<>` for dynamic platform list
