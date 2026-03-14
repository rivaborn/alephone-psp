# Source_Files/Lua/lua_player.cpp

## File Purpose
Implements Lua script bindings for player-related gameplay systems in Aleph One (Marathon game engine). Exposes player data (position, velocity, items, weapons), action input, cameras, HUD overlays, compass, and global game/music state to Lua scripts. Also provides ~50 backward-compatibility wrapper functions for legacy script APIs.

## Core Responsibilities
- Bind player state (position, velocity, energy, weapons, kills) to Lua with type-safe getters/setters
- Manage action flags (input state) from Lua during idle phase via action queues
- Control camera systems including path-based cinematics with spatial waypoints and orientation keyframes
- Manage player HUD overlays (icons, text, colors) for script-driven UI
- Expose compass/beacon system with directional state and world coordinates
- Control game settings (difficulty, game type, scoring mode, time limits)
- Manage music playback, fading, and validation
- Register all player-related Lua classes with the interpreter at startup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| Lua_Action_Flags | Class (template) | Wraps action flag bit state for a player |
| Lua_Camera, Lua_Cameras | Class/Container (template) | Camera objects and collection for cinematics |
| Lua_Camera_Path_Points, Lua_Camera_Path_Angles | Class (template) | Camera path waypoints and orientation keyframes |
| Lua_Player | Class (template) | Main player object exposing all player properties |
| Lua_Players | Container (template) | Collection of all active players |
| Lua_Player_Items, Lua_Player_Weapons | Class (template) | Player inventory and weapon management |
| Lua_InternalVelocity, Lua_ExternalVelocity | Class (template) | Velocity components (facing-relative and world-space) |
| Lua_Overlay, Lua_Overlays | Class/Container (template) | HUD elements and overlay collection |
| Lua_Player_Compass | Class (template) | Compass beacon and direction state |
| Lua_Game, Lua_Music | Class (template) | Global game state and music system control |
| PlayerSubtable | Base class (template) | Template for player-owned sub-objects storing player_index |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| AngleConvert | const float | file | 360/FULL_CIRCLE; converts engine angles to degrees |
| lua_cameras | vector<lua_camera> | extern | All active camera objects |
| number_of_cameras | int | extern | Camera count |
| use_lua_compass | bool[] | extern | Per-player Lua compass enable flag |
| lua_compass_beacons | world_point2d[] | extern | Beacon coordinates per player |
| lua_compass_states | short[] | extern | Compass direction bits per player |
| mute_lua | bool | extern | Suppress Lua print output |
| local_player_index | int | extern | Index of controlled player (for local-only operations) |
| game_scoring_mode | int | extern | Current game scoring mode |
| game_end_condition | int | extern | Game end state flag |
| lua_random_generator | GM_Random | extern | KISS RNG for Lua |
| Lua_*_Name[] | char[] | file | String identifiers for Lua classes (~20 tables) |
| Lua_*_{Get,Set}[] | luaL_reg[] | file | Method tables for getters/setters (~30 tables) |

## Key Functions / Methods

### Lua_Player_register
- **Signature:** `int Lua_Player_register(lua_State *L)`
- **Purpose:** Initialize all player-related Lua bindings; called once at engine startup
- **Inputs:** Lua state
- **Outputs/Return:** 0
- **Side effects:** Registers ~30 Lua classes/containers/enums; pushes global Game and Music userdata
- **Calls:** L_Class::Register(), L_Container::Register(), L_Enum::Register() (many), Lua_Player_load_compatibility()
- **Notes:** Entry point for entire player Lua API

### Lua_Action_Flags_Get_t / Lua_Action_Flags_Set_t (templates)
- **Purpose:** Get/set individual action flag bits (trigger, weapon cycle, map toggle, microphone)
- **Inputs:** Player index; (set only) boolean value
- **Outputs/Return:** (Get) pushes boolean; returns 1. (Set) returns 0
- **Side effects:** (Set) modifies action queue via GetGameQueue()->modifyActionFlags()
- **Calls:** GetGameQueue()->countActionFlags/peekActionFlags/modifyActionFlags()
- **Notes:** Only accessible when action flags queued (idle phase); error otherwise. Template instantiated per action type.

### Lua_Camera_Activate / Lua_Camera_Path_Points_New / Lua_Camera_Path_Angles_New
- **Purpose:** Activate camera for player; add spatial waypoint; add orientation keyframe
- **Inputs:** Camera index; player index / (x,y,z, polygon, time) / (yaw, pitch, time)
- **Outputs/Return:** 0
- **Side effects:** Reset camera state / append to path_points / append to path_angles vectors
- **Notes:** Coordinates scaled by WORLD_ONE; angles divided by AngleConvert; only affects local player

### Lua_Player_Items_Get / Lua_Player_Items_Set
- **Purpose:** Get/set player item count; handles weapon/ball special logic
- **Inputs:** Player index; item type; (set) new count
- **Outputs/Return:** (Get) pushes count; returns 1. (Set) returns 0
- **Side effects:** (Set) calls try_and_add_player_item(), destroy_players_ball(), mark_player_inventory_as_dirty(), select_next_best_weapon()
- **Notes:** Treats NONE sentinel as 0; validates count >= 0; special handling for balls and weapons

### Lua_Player_Get_* / Lua_Player_Set_* (many properties)
- **Purpose:** Access/modify player properties (x, y, z, energy, oxygen, weapons, kills, color, team, direction, etc.)
- **Inputs:** Player index; (set) new value
- **Outputs/Return:** (Get) pushes value; returns 1. (Set) returns 0
- **Side effects:** (Set) marks displays dirty; updates network stats; applies physics recalculation for direction/elevation
- **Notes:** Some operations local-player-only; clamping applied to energy/oxygen

### Lua_Game_Get_* / Lua_Game_Set_* (difficulty, scoring_mode, type, kill_limit, etc.)
- **Purpose:** Expose/modify global game state
- **Inputs:** (Set only) new value
- **Outputs/Return:** (Get) pushes value; returns 1. (Set) returns 0
- **Side effects:** (Set) modifies game_end_condition or game_scoring_mode
- **Calls:** Reads from dynamic_world->game_information

### Lua_Game_Better_Random / Lua_Game_Global_Random / Lua_Game_Local_Random
- **Purpose:** Generate random numbers with optional modulo
- **Inputs:** (Optional) max value
- **Outputs/Return:** Pushes random value
- **Calls:** lua_random_generator.KISS() or ::global_random() or ::local_random()

### Lua_Music_Play / Lua_Music_Stop / Lua_Music_Fade / Lua_Music_Clear
- **Purpose:** Control music playback (load, play, stop, fade out, validate files)
- **Inputs:** File paths (Play accepts variadic); (Fade) duration in seconds
- **Outputs/Return:** 0 or 1
- **Side effects:** Modifies Music::instance() state
- **Calls:** Music::instance()->PushBackLevelMusic(), PreloadLevelMusic(), ClearLevelMusic(), FadeOut(), StopLevelMusic()

### Lua_Overlay_* (Fill_Icon, Set_Icon, Set_Text, Set_Text_Color, Clear)
- **Purpose:** Control per-overlay HUD elements for local player
- **Inputs:** Overlay index; (icon) string; (text) string; (color) color enum
- **Outputs/Return:** 0
- **Side effects:** Calls SetScriptHUDIcon(), SetScriptHUDText(), SetScriptHUDColor(), SetScriptHUDSquare()
- **Notes:** Only affects local_player_index

### Lua_Player_load_compatibility
- **Signature:** `static void Lua_Player_load_compatibility(lua_State *L)`
- **Purpose:** Load ~50 legacy Lua wrapper functions (e.g., accelerate_player, set_player_color)
- **Inputs:** Lua state
- **Side effects:** Defines global Lua functions wrapping new API
- **Calls:** luaL_loadbuffer(), lua_pcall()
- **Notes:** Embeds compatibility_script string; ensures old scripts still work

## Control Flow Notes

**Initialization:** Lua_Player_register() runs once at engine startup to populate Lua VM with all bindings.

**Per-Frame:** No direct per-frame logic. Lua scripts invoke these functions during idle callbacks; action flags are read/written only when player_index is idle (synchronized with game loop's action queue).

**Cinematics:** Camera waypoints and angles are populated via Lua; playback occurs in game update (not here).

**Shutdown:** No explicit cleanup; relies on Lua VM destruction.

## External Dependencies

Notable includes: ActionQueues.h, lua_templates.h, lua_script.h, map.h, player.h, Music.h, Crosshairs.h, fades.h, game_window.h, interface.h, SoundManager.h, ViewControl.h, Random.h

Key external functions: get_player_data(), try_and_add_player_item(), destroy_players_ball(), select_next_best_weapon(), GetGameQueue(), Crosshairs_IsActive/SetActive(), SetScriptHUDIcon/Text/Color/Square(), Music::instance(), instantiate_physics_variables(), mark_shield_display_as_dirty(), draw_panels()
