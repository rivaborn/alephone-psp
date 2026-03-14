# Subsystem Overview

## Purpose
Lua 5.1 scripting layer enabling gameplay mods and cinematics. Provides a virtual machine embedded in the game engine that executes user scripts, exposes game world entities (monsters, items, players, map geometry) to Lua via C++ template bindings, and invokes Lua callbacks in response to game events (damage, item pickup, switch activation). Bridges the PSP-constrained game loop with extensible Lua-based AI, cinematics, and event logic.

## Key Files
| File | Role |
|------|------|
| `lua_script.h` / `lua_script.cpp` | Lua integration layer; script lifecycle, event callback dispatch, game constant registration |
| `lua_templates.h` | Core C++ template infrastructure for binding game objects to Lua (L_Class, L_Container, L_Enum wrappers) |
| `lua.h` | Lua 5.1 public C API; stack manipulation, function calls, type system |
| `language_definition.h` | Game constants mapping (item types, monster types, damage types, game modes) for Lua scripts |
| `lua_map.h` / `lua_map.cpp` | Bindings for map geometry (polygons, lines, sides, platforms, lights, media/liquids) |
| `lua_monsters.h` / `lua_monsters.cpp` | Monster/enemy bindings; creation, targeting, pathfinding, damage |
| `lua_objects.h` / `lua_objects.cpp` | Bindings for map objects (effects, items, scenery) |
| `lua_player.h` / `lua_player.cpp` | Player state bindings (position, velocity, weapons, health, camera control, HUD) |
| `lua_projectiles.h` / `lua_projectiles.cpp` | Projectile bindings; properties, creation, ownership |
| `lua_mnemonics.h` | String-to-integer lookup tables for game constants |
| `lauxlib.h` / `lualib.h` | Lua auxiliary and standard library APIs |
| `lopcodes.h` / `lcode.h` | VM bytecode instruction format and code generation |
| `lparser.h` | Parser interface for compiling source to bytecode |
| `lstate.h` | VM execution state (call stack, threads, global state, garbage collection) |
| `ldo.h` / `lvm.h` | Stack management and bytecode execution engine |

## Core Responsibilities
- **Script loading and lifecycle**: Load Lua scripts from buffers, initialize VM state, register bindings, execute scripts, cleanup on shutdown
- **Game entity exposure**: Wrap indexed game objects (monsters, items, projectiles, players, map elements) as Lua-accessible classes via template machinery
- **Event callbacks**: Dispatch game events (player damage, item pickup, monster death, switch activation, screen fades) to registered Lua handlers
- **Game constant tables**: Register item types, monster types, damage types, game modes, effects, and other Pfhortran mnemonics as Lua globals
- **Player input/control**: Expose action queues and input state to Lua; allow Lua to queue player actions (weapon selection, movement) during idle phases
- **Camera and cinematics**: Provide Lua control over camera positions, orientations, and path-based cutscene cameras with spatial/angular keyframes
- **HUD and UI**: Allow Lua to control HUD overlays (icons, text, colors, compass beacons) and screen effects (fades, palette adjustments)
- **Entity invalidation**: Invalidate Lua references when game objects (monsters, projectiles) are destroyed to prevent dangling pointers
- **Audio/music integration**: Expose music playback, validation, and volume control to Lua scripts

## Key Interfaces & Data Flow

**Exposes to game engine:**
- `L_Call_*` event callback functions (L_Call_MonsterKilled, L_Call_ItemPickedUp, L_Call_PlayerDamaged, etc.)
- `Lua_Player::Push()`, `Lua_Monster::Push()`, `Lua_Polygon::Push()` for pushing game objects onto Lua stack
- Script muting and HUD control flags
- Camera and action queue state for Lua-driven gameplay

**Consumes from game engine:**
- `monsters.h`, `items.h`, `projectiles.h`, `player.h`, `map.h` (entity data structures and accessor functions)
- `render.h`, `screen.h`, `vbl.h` (rendering pipeline)
- `Music.h`, `SoundManager.h` (audio subsystems)
- `platforms.h`, `lightsource.h`, `computer_interface.h` (map-level features)
- `physics_models.h`, `Random.h` (physics and randomness)
- `Crosshairs.h`, `ViewControl.h`, `fades.h` (camera, HUD, effects)

**Consumes from Lua core:**
- Lua 5.1 VM (lua.h, lualib.h, lauxlib.h)
- Bytecode format and instruction set (lopcodes.h, lcode.h, lundump.h)

## Runtime Role
- **Initialization**: Called during game startup to create Lua state, register all bindings, and load scripts
- **Frame update**: During each game frame, `L_Call_Frame` invoked to run Lua frame logic; player action queues polled if Lua code queued input
- **Event dispatch**: Throughout game loop, Lua callbacks fired synchronously when monsters damage, items are picked up, switches activate, players die, etc.
- **Entity cleanup**: When monsters/projectiles/objects destroyed, Lua references invalidated via Lua_Monster::Clear(), Lua_Projectile::Clear(), etc. to prevent use-after-free
- **Shutdown**: Script cleanup and Lua state destruction on game exit

## Notable Implementation Details
- **Template binding pattern** (`lua_templates.h`): L_Class<Index> wraps indexed entity containers; L_Enum wraps numeric mnemonics; L_Container provides iteration and array-like Lua access; metamethods (__index, __newindex, __tostring, __eq) route Lua operations to C++ callbacks
- **Mnemonic-based lookups**: `lua_mnemonics.h` defines sentinel-terminated stringΓåÆinteger arrays for game constants (e.g., `Lua_ItemType_Mnemonics`, `Lua_DamageType_Mnemonics`); enables Lua scripts to use readable names instead of magic numbers
- **Instance caching**: L_Class maintains a Lua registry cache to ensure object identity: `Monsters[5]` returns same Lua object across multiple Lua calls referencing the same monster index
- **Validation predicates**: Template containers accept pluggable `Valid()` and `Length()` callbacks (via `boost::function`) to enforce constraints (e.g., monster index must be in range, must not be deleted)
- **Backward-compatibility layer**: Each Lua binding module (lua_monsters.cpp, lua_objects.cpp, lua_player.cpp) provides ~50 legacy wrapper functions (e.g., `_get_monster_type()`) to support older script APIs alongside modern Lua class methods
- **Conditional compilation**: All Lua code guarded by `#ifdef HAVE_LUA` to allow engine builds without scripting on resource-constrained platforms (including PSP)
- **Boost integration**: Uses `boost::bind` for dynamic container size callbacks (e.g., number of active monsters varies)
