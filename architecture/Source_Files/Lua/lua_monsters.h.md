# Source_Files/Lua/lua_monsters.h

## File Purpose
Defines the Lua scripting interface for monsters in the game engine. Provides type aliases that wrap C++ monster data structures as Lua-accessible classes and containers, allowing Lua scripts to query and interact with game monsters.

## Core Responsibilities
- Define Lua bindings for individual monster instances (`Lua_Monster`)
- Define Lua bindings for the monsters collection (`Lua_Monsters`)
- Define Lua bindings for monster action enumerations (`Lua_MonsterAction`)
- Register all monster-related Lua types and functions with the Lua runtime
- Serve as the public interface between C++ monster management and Lua scripting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Monster` | typedef | Single monster instance wrapper (class type with index) |
| `Lua_Monsters` | typedef | Container/collection of all monsters (enumerable in Lua) |
| `Lua_MonsterAction` | typedef | Enumeration of monster action constants (e.g., attacking, dying) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Monster_Name` | `extern char[]` | extern | String identifier ("monster") for Lua type registration |
| `Lua_Monsters_Name` | `extern char[]` | extern | String identifier ("Monsters") for Lua collection registration |
| `Lua_MonsterAction_Name` | `extern char[]` | extern | String identifier ("monster_action") for Lua enum registration |

## Key Functions / Methods
### Lua_Monsters_register
- **Signature:** `int Lua_Monsters_register(lua_State *L)`
- **Purpose:** Register all monster-related Lua types, classes, and methods with the Lua state
- **Inputs:** Lua state pointer
- **Outputs/Return:** Status code (0 for success, non-zero for error, per Lua convention)
- **Side effects:** Modifies Lua registry; adds global table "Monsters" and related functions to Lua environment
- **Calls:** (Defined elsewhere in implementation file; not visible here)
- **Notes:** Likely calls template methods `Register()` on `Lua_Monster`, `Lua_Monsters`, and `Lua_MonsterAction` to set up getters, setters, and iteration

## Control Flow Notes
This file is part of the game engine's scripting initialization phase. It is included when `HAVE_LUA` is defined and provides the bridge between the engine's C++ monster system (see `monsters.h`) and the Lua runtime. The registration function is typically called during engine startup to expose monster functionality to Lua scripts.

## External Dependencies
- **Notable includes:**
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua C API (wrapped in `extern "C"`)
  - `map.h` ΓÇô Game world/map structures (monster location context)
  - `monsters.h` ΓÇô Engine's C++ monster data structures and functions
  - `lua_templates.h` ΓÇô Template classes for Lua bindings (`L_Class`, `L_Container`, `L_Enum`)
- **External symbols used but not defined here:**
  - Template implementations from `lua_templates.h` (L_Class, L_Container, L_Enum)
  - Monster definitions and game world state from `monsters.h` and `map.h`
  - Lua C API functions (lua_State, luaL_Reg, etc.)
