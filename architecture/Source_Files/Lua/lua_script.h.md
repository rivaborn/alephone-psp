# Source_Files/Lua/lua_script.h

## File Purpose
Declares the C/C++ interface for Lua script integration in a game engine (Aleph One). Provides event callbacks, script lifecycle management, and game state queries to allow Lua scripts to respond to and influence gameplay.

## Core Responsibilities
- **Lifecycle management**: Load, execute, and close Lua scripts
- **Event callbacks**: Dispatch game events (frame updates, player actions, combat, environmental interactions) to Lua via `L_Call_*` functions
- **Entity lifecycle**: Notify Lua when monsters, projectiles, and objects are invalidated
- **Game state exposure**: Provide query functions for camera control, action queues, scoring modes, and player weapon state
- **Audio/rendering**: Manage Lua-related muting and texture palette access
- **Cutscene cameras**: Support timed camera paths with angle interpolation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `timed_point` | struct | Represents a point in world space with associated polygon and delta time for camera movement |
| `timed_angle` | struct | Stores yaw/pitch angles with delta time for camera rotation sequencing |
| `lua_path` | struct | Defines a camera path as sequences of timed points and angles with current playback indices |
| `lua_camera` | struct | Full camera state including index, path data, elapsed time, and active player |
| Game scoring modes (enum) | enum | Four scoring rule sets: most/least points/time |
| Game end conditions (enum) | enum | Three end conditions: normal, none, or immediate |

## Global / File-Static State
None.

## Key Functions / Methods

### L_Error
- **Signature**: `void L_Error(const char *message)`
- **Purpose**: Report errors from Lua script execution
- **Inputs**: Error message string
- **Outputs/Return**: None
- **Side effects**: Likely logs or displays error to user
- **Calls**: Not inferable
- **Notes**: No indication of whether execution continues

### L_Call_Init / L_Call_Cleanup
- **Signature**: `void L_Call_Init(bool fRestoringSaved)` / `void L_Call_Cleanup()`
- **Purpose**: Initialize Lua environment at startup; clean up at shutdown
- **Inputs**: `fRestoringSaved` indicates whether loading a saved game state
- **Outputs/Return**: None
- **Side effects**: Initializes or tears down Lua interpreter and associated state
- **Calls**: Not inferable
- **Notes**: Lifecycle bookends; called once per session

### L_Call_Idle / L_Call_PostIdle
- **Signature**: `void L_Call_Idle()` / `void L_Call_PostIdle()`
- **Purpose**: Frame update callbacks; `Idle` runs before game logic, `PostIdle` after
- **Inputs**: None
- **Outputs/Return**: None
- **Side effects**: Executes per-frame Lua logic; may modify game state
- **Calls**: Not inferable
- **Notes**: Called every frame; timing critical for real-time gameplay behavior

### L_Call_\* (Event Callbacks)
- **Pattern**: `void L_Call_<Event>(short <index>, ...)`
- **Purpose**: Dispatch specific game events to Lua (refueling, switches, terminal access, combat, projectiles, items)
- **Examples**:
  - `L_Call_Player_Killed(short player_index, short aggressor_player_index, short action, short projectile_index)` ΓÇö Player death with aggressor context
  - `L_Call_Projectile_Detonated(short type, short owner_index, short polygon, world_point3d location)` ΓÇö Projectile impact event
  - `L_Call_Monster_Killed(short monster_index, short aggressor_player_index, short projectile_index)` ΓÇö Monster death
- **Side effects**: Lua scripts respond and may trigger game state changes or action queue modifications
- **Notes**: Events are dispatched conditionally; Lua script subscription is implicit

### LoadLuaScript / RunLuaScript / CloseLuaScript
- **Signature**: `bool LoadLuaScript(const char *buffer, size_t len)` / `bool RunLuaScript()` / `void CloseLuaScript()`
- **Purpose**: Load script from buffer, execute loaded script, close and unload script
- **Inputs**: `buffer` and `len` for loading; none for execution/closure
- **Outputs/Return**: `bool` success/failure for load and run; void for close
- **Side effects**: Script loading/unloading, Lua environment state changes
- **Calls**: Not inferable
- **Notes**: Three-phase model (load ΓåÆ run ΓåÆ close); typical script lifecycle

### L_Invalidate_\* (Entity Notifications)
- **Signature**: `void L_Invalidate_Monster(short monster_index)` / `void L_Invalidate_Projectile(short projectile_index)` / `void L_Invalidate_Object(short object_index)`
- **Purpose**: Notify Lua that a game entity has been destroyed and should no longer be referenced
- **Inputs**: Entity index
- **Outputs/Return**: None
- **Side effects**: Lua removes stale references to prevent crashes on invalid entity access
- **Calls**: Not inferable
- **Notes**: Prevents Lua scripts from dereferencing freed entities

### GetLuaActionQueues
- **Signature**: `ActionQueues* GetLuaActionQueues()`
- **Purpose**: Retrieve the action queue manager for Lua-controlled player input
- **Inputs**: None
- **Outputs/Return**: Pointer to `ActionQueues` instance
- **Side effects**: None; read-only query
- **Calls**: Not inferable
- **Notes**: Allows Lua to enqueue player actions (movement, weapon fire, etc.)

### LuaTexturePalette\* / MarkLuaCollections
- **Signature**: `void LuaTexturePaletteClear()` / `int LuaTexturePaletteSize()` / `shape_descriptor LuaTexturePaletteTexture(size_t)` / `int LuaTexturePaletteSelected()` / `void MarkLuaCollections(bool active)`
- **Purpose**: Manage custom texture palette for Lua use; mark sprite collections as active/inactive
- **Inputs**: Index for palette texture query; bool for collection marking
- **Outputs/Return**: Palette size, descriptor, or selected index; void for marking
- **Side effects**: Modifies rendering state if collections are toggled
- **Calls**: Not inferable
- **Notes**: Supports visual customization and collection culling

### LuaPlayerCanWieldWeapons
- **Signature**: `bool LuaPlayerCanWieldWeapons(short player_index)`
- **Purpose**: Query whether a player can wield weapons (e.g., zombie state)
- **Inputs**: Player index
- **Outputs/Return**: `bool` capability flag
- **Side effects**: None; read-only query
- **Calls**: Not inferable
- **Notes**: Used to enforce gameplay constraints set by Lua

### GetLuaScoringMode / GetLuaGameEndCondition
- **Signature**: `int GetLuaScoringMode()` / `int GetLuaGameEndCondition()`
- **Purpose**: Query current game scoring and end-condition modes
- **Inputs**: None
- **Outputs/Return**: Enum values (indices into `_game_of_most_points`, `_game_normal_end_condition`, etc.)
- **Side effects**: None; read-only queries
- **Calls**: Not inferable
- **Notes**: Allows Lua to adapt behavior to game ruleset

---

### Trivial Helpers (summarized)
- `ExecuteLuaString(const std::string&)` ΓÇö Execute arbitrary Lua code string
- `LoadSoloLua()` ΓÇö Load campaign/solo mode Lua script
- `ToggleLuaMute()` / `ResetLuaMute()` ΓÇö Audio state control
- `UseLuaCameras()` ΓÇö Check if Lua controls player viewpoint

## Control Flow Notes
**Initialization ΓåÆ Frame Loop ΓåÆ Shutdown**
1. **Startup**: `L_Call_Init()` initializes Lua, script loaded via `LoadLuaScript()`, then `RunLuaScript()`
2. **Each Frame**: `L_Call_Idle()` runs, game logic executes, `L_Call_PostIdle()` runs
3. **Event Handling**: On any significant game event (player damage, item pickup, combat), the corresponding `L_Call_*` is dispatched
4. **Entity Cleanup**: When monsters/projectiles/objects are destroyed, `L_Invalidate_*` notifies Lua
5. **Shutdown**: `CloseLuaScript()` and `L_Call_Cleanup()` clean up

## External Dependencies
- **cseries.h**: Core C++ utilities (types, macros, standard library wrappers)
- **world.h**: World coordinates (`world_point3d`, `world_distance`, angle types)
- **ActionQueues.h**: Player input queue management (`ActionQueues` class)
- **shape_descriptors.h**: Texture/sprite descriptor type (`shape_descriptor`)
