# Source_Files/GameWorld/devices.cpp

## File Purpose
Manages interactive control panels and device switches in the game worldΓÇödoors, refuel stations, save points, computer terminals, and various trigger switches. Handles player interaction, panel state management, texture updates, and networked save coordination.

## Core Responsibilities
- Define and initialize control panel types with audio/visual properties
- Detect and handle player action-key activation of nearby panels
- Update continuous interactions (oxygen/shield recharge, auto-save polling)
- Toggle panel states and cascade effects to linked lights/platforms
- Validate switch accessibility (lighting, item requirements, projectile-only)
- Play panel sounds and update textures based on state
- Coordinate pattern-buffer saves in networked games
- Parse and apply XML configuration overrides for panel properties

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `control_panel_definition` | struct | Panel type: class, textures, sounds, item cost |
| `control_panel_settings_definition` | struct | Runtime settings: reach distance, recharge rates/limits |
| `XML_CPSoundParser` | class | XML parser for panel sounds |
| `XML_ControlPanelParser` | class | XML parser for individual panel overrides |
| `XML_ControlPanelsParser` | class | XML parser for global settings container |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `control_panel_definitions[]` | array of struct | static | Hardcoded panel types; modified by XML |
| `control_panel_settings` | struct | static | Activation ranges, recharge rates |
| `original_control_panel_definitions` | pointer | static | Backup for XML reset |
| `original_control_panel_settings` | pointer | static | Backup for XML reset |
| `CPSoundParser`, `ControlPanelParser`, `ControlPanelsParser` | parser instances | static | XML element parsers |

## Key Functions / Methods

### initialize_control_panels_for_level()
- **Signature**: `void initialize_control_panels_for_level(void)`
- **Purpose**: Set initial panel states based on map data (tag switches, light switches, platform switches)
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Modifies all `side_data` control panel status and texture
- **Calls**: `get_control_panel_definition()`, `set_control_panel_texture()`, light/platform accessors
- **Notes**: Runs once per level load; reads platform/light states to match panel visual

### update_control_panels()
- **Signature**: `void update_control_panels(void)`
- **Purpose**: Handle continuous interactions (oxygen/shield recharge, pattern-buffer save timing)
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Modifies player suit energy/oxygen; updates panel textures; triggers saves or stops activating sound
- **Calls**: `get_recharge_status()`, `set_control_panel_texture()`, `play_control_panel_sound()`, `somebody_save_full_auto()`
- **Notes**: Per-frame; only processes players with `control_panel_side_index != NONE`; includes netgame double-click detection for saves

### update_action_key()
- **Signature**: `void update_action_key(short player_index, bool triggered)`
- **Purpose**: Handle player action-key press; find and interact with targets
- **Inputs**: Player index, trigger state
- **Outputs/Return**: None
- **Side effects**: Calls platform or panel handlers; may change game state
- **Calls**: `find_action_key_target()`, `player_touch_platform_state()`, `change_panel_state()`
- **Notes**: Dispatches to appropriate handler based on target type enum

### find_action_key_target()
- **Signature**: `short find_action_key_target(short player_index, world_distance range, short *target_type)`
- **Purpose**: Raycast from player in facing direction; find nearest platform or panel within range
- **Inputs**: Player index, max range, output type pointer
- **Outputs/Return**: Object index (platform/side index) or NONE
- **Side effects**: Sets `*target_type` to `_target_is_platform`, `_target_is_control_panel`, or `_target_is_unrecognized`
- **Calls**: `ray_to_line_segment()`, `find_line_crossed_leaving_polygon()`, `line_side_has_control_panel()`, `switch_can_be_toggled()`, `line_is_within_range()`
- **Notes**: Walks through polygon chain; respects platform and panel activation ranges

### change_panel_state()
- **Signature**: `static void change_panel_state(short player_index, short panel_side_index)`
- **Purpose**: Toggle or activate a control panel; handle all interaction types
- **Inputs**: Player and side indices
- **Outputs/Return**: None
- **Side effects**: Modifies panel status, player state (items, control_panel_side_index), suit energy, calls save functions, updates linked lights/platforms, plays sounds
- **Calls**: `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `set_light_status()`, `try_and_change_platform_state()`, `enter_computer_interface()`, `save_game()`, Lua hooks
- **Notes**: Large switch on panel class; handles oxygen/shield refuel, computer terminal, tag/light/platform switches, pattern buffer (save), and item consumption

### try_and_toggle_control_panel()
- **Signature**: `void try_and_toggle_control_panel(short polygon_index, short line_index)`
- **Purpose**: Trigger panel from monster/script (non-player interaction)
- **Inputs**: Polygon and line indices from environment
- **Outputs/Return**: None
- **Side effects**: Toggles panel state, updates linked devices, plays sounds; includes Lua hooks
- **Calls**: `find_adjacent_side()`, `switch_can_be_toggled()`, panel toggle logic, `play_control_panel_sound()`, Lua hooks
- **Notes**: Parallel to `change_panel_state()` but for environmental triggers; no player state changes

### switch_can_be_toggled()
- **Signature**: `static bool switch_can_be_toggled(short side_index, bool player_hit)`
- **Purpose**: Validate switch activation constraints
- **Inputs**: Side index, whether hit by player (vs. projectile)
- **Outputs/Return**: Boolean validity
- **Side effects**: May destroy switch if flagged; plays unusable sound on rejection
- **Calls**: `get_light_intensity()`, `play_control_panel_sound()`
- **Notes**: Checks light requirement (must be >75%), item cost (fails if player_hit and item required), projectile-only restriction, destructibility

### set_control_panel_texture()
- **Signature**: `void set_control_panel_texture(struct side_data *side)`
- **Purpose**: Update side texture to reflect current panel state
- **Inputs**: Pointer to side data
- **Outputs/Return**: None
- **Side effects**: Changes `side->primary_texture.texture`
- **Calls**: `get_control_panel_definition()`, shape descriptor macros
- **Notes**: Simple: selects active or inactive shape based on status bit

### line_side_has_control_panel()
- **Signature**: `bool line_side_has_control_panel(short line_index, short polygon_index, short *side_index_with_panel)`
- **Purpose**: Check if a line/polygon pair has a panel; retrieve side index
- **Inputs**: Line and polygon indices, output pointer
- **Outputs/Return**: Boolean; sets `*side_index_with_panel` if found
- **Side effects**: None
- **Calls**: `get_line_data()`, `get_side_data()`, macro checks
- **Notes**: Handles both clockwise and counterclockwise polygon ownership

### line_is_within_range()
- **Signature**: `static bool line_is_within_range(short monster_index, short line_index, world_distance range)`
- **Purpose**: Check distance from monster to line midpoint, weighted by vertical separation
- **Inputs**: Monster and line indices, max distance
- **Outputs/Return**: Boolean
- **Side effects**: None
- **Calls**: `calculate_line_midpoint()`, `get_monster_dimensions()`, `isqrt()`
- **Notes**: Vertical distance component is weighted by `ReachHorizontal` setting

### somebody_save_full_auto()
- **Signature**: `static void somebody_save_full_auto(player_data* inWhoSaved, bool inOverwrite)`
- **Purpose**: Execute full auto-save when player uses pattern buffer
- **Inputs**: Player pointer, overwrite flag (netgame double-click)
- **Outputs/Return**: None
- **Side effects**: Clears player's panel interaction flag, updates save tick, calls save functions, broadcasts message in netgames
- **Calls**: `play_control_panel_sound()`, `save_game_full_auto()` or `save_game()`, `screen_printf()`
- **Notes**: Prevents re-entrancy; updates tick tracking to suppress rapid re-saves

### get_recharge_status()
- **Signature**: `static bool get_recharge_status(short side_index)`
- **Purpose**: Check if any player is currently using a recharge panel
- **Inputs**: Side index
- **Outputs/Return**: Boolean
- **Side effects**: None
- **Calls**: None
- **Notes**: Simple iteration over `players[]` checking `control_panel_side_index`

## Control Flow Notes
**Initialization**: `initialize_control_panels_for_level()` called on level entry; reads map side flags and initializes panel states.

**Per-Frame Update**: `update_control_panels()` handles standing-on-panel refueling and auto-save cooldown; `update_action_key()` called on action button and performs raycast + dispatch.

**Interaction Chain**: Player presses action ΓåÆ raycast finds target ΓåÆ `change_panel_state()` toggles panel ΓåÆ cascades to lights/platforms ΓåÆ sounds and textures update ΓåÆ Lua hooks fire.

**Environment Triggers**: `try_and_toggle_control_panel()` used when monsters or level events trigger panels via polygon/line references.

## External Dependencies
- **map.h**: Side, line, polygon data; map geometry queries
- **player.h**: Player state, items, control panel side index
- **monsters.h**: Monster dimensions and visibility  
- **platforms.h**: Platform state modification  
- **SoundManager.h**: Sound playback API  
- **computer_interface.h**: Terminal entry point  
- **lightsource.h** (implicit): Light status queries  
- **lua_script.h**: Script hook functions (`L_Call_*`)
- **Defined elsewhere**: `dynamic_world`, `map_sides`, `players`, `local_player`, save/load functions, light/platform control functions, XML parser base class
