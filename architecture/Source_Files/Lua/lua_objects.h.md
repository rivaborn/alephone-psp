# Source_Files/Lua/lua_objects.h

## File Purpose
Declares Lua language bindings for game map objects (effects, items, scenery, sounds) in Aleph One (Marathon engine port). Uses template-based wrappers to expose C++ game entities as Lua objects with introspection and iteration support.

## Core Responsibilities
- Define typedefs for Lua class and container wrappers for game object types
- Export external string constants used as Lua metatable identifiers
- Declare the registration function that initializes all object bindings in a Lua state
- Provide separate class and container interfaces for singular objects and object collections
- Support both regular classes and enum-like types with mnemonic string lookups

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Effect` | typedef (`L_Class<Lua_Effect_Name>`) | Wrapper for individual in-game effect object |
| `Lua_Effects` | typedef (`L_Container<...>`) | Container for iteration over all effects |
| `Lua_EffectType` | typedef (`L_Enum<...>`) | Enum wrapper for effect type constants |
| `Lua_EffectTypes` | typedef (`L_EnumContainer<...>`) | Container for effect type enumeration with mnemonic lookup |
| `Lua_Item` | typedef (`L_Class<...>`) | Wrapper for individual item object |
| `Lua_Items` | typedef (`L_Container<...>`) | Container for iterating items |
| `Lua_ItemType` | typedef (`L_Enum<...>`) | Enum wrapper for item type constants |
| `Lua_ItemTypes` | typedef (`L_EnumContainer<...>`) | Container for item type enumeration |
| `Lua_Scenery` | typedef (`L_Class<...>`) | Wrapper for scenery/static object |
| `Lua_Sceneries` | typedef (`L_Container<...>`) | Container for scenery objects |
| `Lua_Sound` | typedef (`L_LazyEnum<...>`) | Lazy-loading enum for sound identifiers (loads on first lookup) |
| `Lua_Sounds` | typedef (`L_EnumContainer<...>`) | Container for sound enumeration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Effect_Name` | `extern char[]` | global | Metatable identifier string for Effect class (value: `"effect"`) |
| `Lua_Effects_Name` | `extern char[]` | global | Metatable identifier for Effects container (value: `"Effects"`) |
| `Lua_EffectType_Name` | `extern char[]` | global | Metatable identifier for EffectType enum (value: `"effect_type"`) |
| `Lua_EffectTypes_Name` | `extern char[]` | global | Metatable identifier for EffectTypes container (value: `"EffectTypes"`) |
| `Lua_Item_Name` | `extern char[]` | global | Metatable identifier for Item class (value: `"item"`) |
| `Lua_Items_Name` | `extern char[]` | global | Metatable identifier for Items container (value: `"Items"`) |
| `Lua_ItemType_Name` | `extern char[]` | global | Metatable identifier for ItemType enum (value: `"item_type"`) |
| `Lua_ItemTypes_Name` | `extern char[]` | global | Metatable identifier for ItemTypes container (value: `"ItemTypes"`) |
| `Lua_Scenery_Name` | `extern char[]` | global | Metatable identifier for Scenery class (value: `"scenery"`) |
| `Lua_Sceneries_Name` | `extern char[]` | global | Metatable identifier for Sceneries container (value: `"Scenery"`) |
| `Lua_Sound_Name` | `extern char[]` | global | Metatable identifier for Sound lazy enum (value: `"sound"`) |
| `Lua_Sounds_Name` | `extern char[]` | global | Metatable identifier for Sounds container (value: `"Sounds"`) |

## Key Functions / Methods

### Lua_Objects_register
- **Signature:** `int Lua_Objects_register(lua_State *L);`
- **Purpose:** Initialize and register all game object bindings (effects, items, scenery, sounds) into the Lua environment.
- **Inputs:** `lua_State *L` ΓÇö active Lua interpreter instance
- **Outputs/Return:** Return code (typically 0 for success, non-zero for error) following Lua C API convention
- **Side effects (global state, I/O, alloc):** 
  - Modifies Lua registry to define metatables for all object types
  - Creates Lua global tables for object containers (`Effects`, `Items`, `Scenery`, `Sounds`)
  - Allocates Lua stack space and registry entries for class metadata
- **Calls (direct calls visible in this file):** Not defined in this file; implementation in corresponding `.cpp` file (not shown)
- **Notes (invariants, edge cases):** 
  - Must be called exactly once after `lua_State` creation
  - Depends on external string constants being initialized
  - Implementation likely uses `L_Class<>::Register()` and `L_Container<>::Register()` templates from lua_templates.h

## Control Flow Notes
This header participates in **engine initialization**. A typical flow would be:
1. Lua state created (`lua_newstate`)
2. `Lua_Objects_register(L)` called to expose game objects to scripts
3. Game Lua scripts loaded and executed; they can now access and iterate Effects, Items, Scenery, Sounds collections
4. During game updates, C++ code may push/pop individual objects onto the Lua stack for script callbacks

## External Dependencies
- **Lua 5.1 API:** lua.h, lauxlib.h, lualib.h ΓÇö Lua interpreter and auxiliary library functions
- **Game world:** items.h (item type definitions), map.h (map/world geometry structures) 
- **Lua binding templates:** lua_templates.h (provides `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`, `L_LazyEnum`)
- **Platform utilities:** cseries.h (core series library with platform abstractions)
