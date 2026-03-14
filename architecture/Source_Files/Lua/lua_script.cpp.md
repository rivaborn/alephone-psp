# Source_Files/Lua/lua_script.cpp

## File Purpose
Lua script integration layer for a Marathon-like game engine (Aleph One). Manages loading, execution, and cleanup of Lua scripts; registers C++ functions and game constants for Lua; and invokes Lua event callbacks throughout the game lifecycle.

## Core Responsibilities
- Load Lua scripts from buffers and execute them within a Lua state
- Register native C functions exposing game engine capabilities to Lua
- Declare game constants (item types, object counts, damage types, etc.) as Lua globals
- Invoke Lua event triggers at key game moments (player damage, item pickup, switch activation, etc.)
- Provide Lua with direct player control, camera manipulation, and game state queries
- Invalidate Lua object references when game entities (monsters, projectiles, items) are destroyed
- Manage Lua-controlled HUD display, screen fades, and gameplay UI
- Toggle script muting and handle script lifecycle cleanup

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `lua_State` | opaque C struct | Lua virtual machine state |
| `lua_camera` | struct | Path and timing data for scripted camera motion |
| `ActionQueues` | class | Queue for enqueueing player movement/action commands |
| `GM_Random` | struct | Better random number generator seeded for Lua |
| `world_point3d` | struct | 3D world coordinate (x, y, z in world units) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `state` | `lua_State*` | global | Active Lua interpreter instance |
| `lua_loaded` | `bool` | global | True if a script has been loaded into state |
| `lua_running` | `bool` | global | True if the loaded script executed successfully |
| `lua_cameras` | `vector<lua_camera>` | global | Queued camera paths for Lua-controlled cinematic sequences |
| `mute_lua` | `bool` | static | Suppress on-screen Lua error messages |
| `use_lua_compass` | `bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | global | Per-player flag to enable compass beacons |
| `can_wield_weapons` | `bool[MAXIMUM_NUMBER_OF_NETWORK_PLAYERS]` | global | Per-player flag to permit weapon wielding |
| `game_scoring_mode` | `int` | global | Current scoring rule (e.g., `_game_of_most_points`) |
| `game_end_condition` | `int` | global | End-game trigger condition |
| `lua_random_generator` | `GM_Random` | global | Seeded RNG for `better_random()` Lua function |
| `sLuaActionQueues` | `ActionQueues*` | static | Singleton queue for player input enqueuing |
| `sLuaNetscriptLoaded` | `bool` | static | Tracks netscript vs. user script loading conflict |
| `gLoadingLuaNetscript` | `bool` | global | Flag set when netscript system loads a script |
| `numScriptsLoaded` | `int` | static | Count of scripts loaded in current Lua state |
| `old_size` | `short` | static | Saved screen size before HUD hide/show |

## Key Functions / Methods

### LoadLuaScript
- **Signature:** `bool LoadLuaScript(const char *buffer, size_t len)`
- **Purpose:** Load a Lua script buffer into the global Lua state; initialize state and standard libraries on first load.
- **Inputs:** `buffer` (Lua source code), `len` (buffer size in bytes)
- **Outputs/Return:** `true` if load successful, `false` if parse/load error
- **Side effects:** Allocates/initializes `lua_State`, registers functions/constants on first call; logs warnings via `logWarning()` for various Lua errors; sets `lua_loaded`, increments `numScriptsLoaded`
- **Calls:** `lua_open()`, `OpenStdLibs()`, `RegisterLuaFunctions()`, `DeclareLuaConstants()`, `luaL_loadbuffer()`, `logWarning()`
- **Notes:** Does not execute; multiple scripts can be queued before `RunLuaScript()`. Conflicting load of netscript + user script triggers alert dialog.

### RunLuaScript
- **Signature:** `bool RunLuaScript()`
- **Purpose:** Execute all queued Lua scripts and call the optional `init` trigger if present.
- **Inputs:** None
- **Outputs/Return:** `true` if execution succeeded (no runtime errors), `false` if not loaded or execution failed
- **Side effects:** Sets `lua_running`; preserves and restores OpenGL fog and motion-sensor settings; resets compass/weapon flags; calls `lua_pcall()` for each buffered script
- **Calls:** `PreservePreLuaSettings()`, `lua_insert()`, `lua_pcall()`, `RestorePreLuaSettings()`, `L_Call_Init()`
- **Notes:** Scripts are re-ordered on stack before execution; only one Lua state per game session.

### CloseLuaScript
- **Signature:** `void CloseLuaScript()`
- **Purpose:** Clean up the Lua state, restore engine settings, and reset script tracking.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `lua_close()`, clears `lua_cameras`, calls `LuaTexturePaletteClear()`, resets scoring/end-condition to defaults, calls `RestorePreLuaSettings()`
- **Calls:** `lua_close()`, `RestorePreLuaSettings()`, `LuaTexturePaletteClear()`
- **Notes:** Invoked on level exit; must be called before loading a new script.

### RegisterLuaFunctions
- **Signature:** `void RegisterLuaFunctions()`
- **Purpose:** Register all C functions and compatibility triggers in the Lua state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `lua_register()` for engine functions; loads compatibility wrapper code into Lua global `Triggers` table; calls sub-module registration (Lua_Map_register, etc.)
- **Calls:** `lua_register()`, `Lua_Map_register()`, `Lua_Monsters_register()`, `Lua_Objects_register()`, `Lua_Player_register()`, `Lua_Projectiles_register()`, `luaL_loadbuffer()`
- **Notes:** Compatibility layer allows old single-function names (e.g., `init`, `cleanup`) to be wrapped in the `Triggers` table.

### DeclareLuaConstants
- **Signature:** `void DeclareLuaConstants()`
- **Purpose:** Parse language definitions and expose them as global Lua numbers.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Reads from `language_definition.h` via struct array; calls `lua_pushnumber()` and `lua_setglobal()` for each constant and object limit
- **Calls:** (indirectly reads `language_definition.h`), `lua_pushnumber()`, `lua_setglobal()`
- **Notes:** Exposes `MAXIMUM_MONSTERS_PER_MAP`, `MAXIMUM_PROJECTILES_PER_MAP`, `MAXIMUM_OBJECTS_PER_MAP`.

### L_Call_* Trigger Functions (Event Callbacks)
Examples: `L_Call_Init()`, `L_Call_Idle()`, `L_Call_Tag_Switch()`, `L_Call_Player_Killed()`, `L_Call_Projectile_Detonated()`, etc.
- **Signature:** `void L_Call_*(...)`  (arguments vary per trigger)
- **Purpose:** Invoke a named Lua trigger function with game-event data (e.g., player index, damage type, position).
- **Inputs:** Game-specific parameters (player indices, object types, positions, etc.)
- **Outputs/Return:** None
- **Side effects:** Pushes trigger data onto Lua stack; calls `lua_pcall()` if trigger exists; on error, calls `L_Error()` to log
- **Calls:** `L_Get_Trigger()`, `Lua_*::Push()` methods, `L_Call_Trigger()`, `L_Error()`
- **Notes:** Each wraps a common pattern: check trigger existence, push arguments, call via `lua_pcall()`; if trigger missing, silently returns.

### L_Player_Control
- **Signature:** `static int L_Player_Control(lua_State *L)`
- **Purpose:** Lua C function to enqueue player movement/action flags.
- **Inputs:** Lua stack: player index, move type, duration/count, and optional goal coordinates
- **Outputs/Return:** 0 (no Lua return values)
- **Side effects:** Allocates/resizes `action_flags` array; enqueues actions via `GetLuaActionQueues()->enqueueActionFlags()`; may instantiate physics variables if `TIENNOU_PLAYER_CONTROL` enabled
- **Calls:** `GetLuaActionQueues()`, physics update functions (if enabled), `lua_error()`
- **Notes:** Move types include forward/backward/sidestep, turning, looking, and triggers; code has large conditional block guarded by `TIENNOU_PLAYER_CONTROL` (appears incomplete/experimental).

### UseLuaCameras
- **Signature:** `bool UseLuaCameras()`
- **Purpose:** Update active Lua camera and set world view from camera path data.
- **Inputs:** None
- **Outputs/Return:** `true` if a Lua camera is active for local player
- **Side effects:** Modifies `world_view->origin`, `world_view->yaw`, `world_view->pitch`, `world_view->origin_polygon_index`; calls `ShootForTargetPoint()` for ray-casting; increments camera timers and advances path indices
- **Calls:** `FindLinearValue()`, `ShootForTargetPoint()`, `normalize_angle()`
- **Notes:** Handles linear interpolation between camera waypoints and angles; silently skips if no cameras or script not running.

### ExecuteLuaString
- **Signature:** `void ExecuteLuaString(const std::string& line)`
- **Purpose:** Execute a Lua command from the console, optionally printing return value.
- **Inputs:** `line` (Lua code; prefix `=` means print result)
- **Outputs/Return:** None
- **Side effects:** Initializes Lua state if not loaded; calls `lua_pcall()`; prints result to screen via `screen_printf()` if requested
- **Calls:** `lua_open()`, `OpenStdLibs()`, `RegisterLuaFunctions()`, `DeclareLuaConstants()`, `luaL_loadbuffer()`, `lua_pcall()`, `L_Error()`, `screen_printf()`
- **Notes:** Lazy initialization: Lua state created on first console command if not yet loaded by level script.

### FindLinearValue (Utility, 2 overloads)
- **Signature:** `double FindLinearValue(double start, double end, double range, double elapsed)` and `world_point3d FindLinearValue(world_point3d start, world_point3d end, ...)`
- **Purpose:** Interpolate between two values/points given elapsed time within a time range.
- **Inputs:** Start, end, time range, elapsed time (all as `double`)
- **Outputs/Return:** Interpolated value or 3D point
- **Side effects:** None
- **Calls:** None (pure computation)
- **Notes:** Linear interpolation formula: `((end - start) / range) * elapsed + start`; used for camera and animation paths.

### L_Invalidate_Monster, L_Invalidate_Projectile, L_Invalidate_Object
- **Signature:** `void L_Invalidate_*(short index)`
- **Purpose:** Notify Lua that a game entity has been destroyed; invalidate Lua references to prevent dangling pointers.
- **Inputs:** `index` (object/entity index)
- **Outputs/Return:** None
- **Side effects:** Calls corresponding `Lua_*::Invalidate()` method
- **Calls:** `Lua_Monster::Invalidate()`, `Lua_Projectile::Invalidate()`, `Lua_Item::Invalidate()`, `Lua_Effect::Invalidate()`, `Lua_Scenery::Invalidate()`
- **Notes:** Called by engine when objects are destroyed; `L_Invalidate_Object` determines object type and calls appropriate invalidator.

### L_Enable_Player, L_Disable_Player (Lua C Functions)
- **Signature:** `static int L_Enable_Player(lua_State *L)`, `static int L_Disable_Player(lua_State *L)`
- **Purpose:** Toggle "zombie" (disabled) status for a player.
- **Inputs:** Lua stack: player index
- **Outputs/Return:** 0
- **Side effects:** Calls `SET_PLAYER_ZOMBIE_STATUS()` macro
- **Calls:** `get_player_data()`, `SET_PLAYER_ZOMBIE_STATUS()`, `lua_error()`
- **Notes:** No effect if player is dead; validates index before access.

### L_Hide_Interface, L_Show_Interface (Lua C Functions)
- **Signature:** `static int L_Hide_Interface(lua_State *L)`, `static int L_Show_Interface(lua_State *L)`
- **Purpose:** Toggle HUD display for the local player.
- **Inputs:** Lua stack: player index
- **Outputs/Return:** 0
- **Side effects:** Modifies screen size mode; calls `change_screen_mode()` and `draw_panels()`
- **Calls:** `get_screen_mode()`, `GetSizeWithoutHUD()`, `GetSizeWithHUD()`, `change_screen_mode()`, `draw_panels()`, `lua_error()`
- **Notes:** Only affects local player; uses saved `old_size` to restore previous size.

---

## Control Flow Notes
**Lifecycle:**
1. Level load ΓåÆ `LoadLuaScript()` (optional, if MML or netscript system provides script)
2. Game start ΓåÆ `RunLuaScript()` (if script loaded) ΓåÆ invokes `Triggers.init()`
3. Main game loop ΓåÆ frame/idle/post-idle ΓåÆ `L_Call_Idle()`, `L_Call_PostIdle()` each tick
4. Event triggers ΓåÆ player damage, item pickup, switch press, etc. ΓåÆ respective `L_Call_*()` fires corresponding Lua function
5. Level/game exit ΓåÆ `CloseLuaScript()` ΓåÆ invokes `Triggers.cleanup()`

**Trigger Guard Pattern:**  
All `L_Call_*` functions follow: `L_Get_Trigger(name)` ΓåÆ check if Lua function exists in `Triggers` table ΓåÆ push args ΓåÆ `L_Call_Trigger()` ΓåÆ `lua_pcall()`. Silent no-op if trigger absent.

**Camera Integration:**  
`UseLuaCameras()` called each frame from render loop; linearly interpolates position/angle along predefined paths; updates `world_view` directly.

## External Dependencies
**Lua Runtime:**
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.1 interpreter and auxiliary library
- Standard libs: base, table, string, math, debug (IO and loadlib omitted for security)

**Game Engine:**
- `player.h`, `monsters.h`, `projectiles.h`, `items.h`, `world.h` ΓÇö Game object definitions
- `render.h`, `screen.h`, `vbl.h` ΓÇö Rendering and frame timing
- `shell.h`, `interface.h` ΓÇö Game state and UI
- `network.h`, `network_games.h` ΓÇö Multiplayer support
- `lightsource.h`, `platforms.h`, `computer_interface.h` ΓÇö Map features
- `physics_models.h`, `Random.h` ΓÇö Physics and randomness
- `fades.h`, `Music.h`, `SoundManager.h` ΓÇö Audio/visual effects
- `Console.h` ΓÇö Debug console
- `Crosshairs.h`, `ViewControl.h`, `OGL_Setup.h` ΓÇö Camera and rendering config

**Lua Binding Modules (defined elsewhere):**
- `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_projectiles.h` ΓÇö Type/function wrappers for game classes

**Constants:**
- `language_definition.h` ΓÇö Game constants (included with `DONT_REPEAT_DEFINITIONS`)
- `item_definitions.h`, `monster_definitions.h` ΓÇö Object metadata
