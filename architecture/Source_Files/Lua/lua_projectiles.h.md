# Source_Files/Lua/lua_projectiles.h

## File Purpose
Header file defining Lua bindings for the projectiles subsystem. Declares type-alias wrappers around template classes to expose projectile objects and projectile types to Lua scripts. Only compiled when `HAVE_LUA` is defined.

## Core Responsibilities
- Declares Lua class wrapper (`Lua_Projectile`) for individual projectile game objects
- Declares Lua container wrapper (`Lua_Projectiles`) for projectile collections
- Declares Lua enum wrapper (`Lua_ProjectileType`) for projectile type enumerations
- Declares Lua enum container wrapper (`Lua_ProjectileTypes`) for projectile type collections
- Declares the module registration function to initialize Lua bindings
- Provides type-safe bridging between game engine and Lua script layer

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Projectile` | typedef (L_Class template) | Wraps individual projectile object for Lua access |
| `Lua_Projectiles` | typedef (L_Container template) | Collection container allowing Lua to iterate/index projectiles |
| `Lua_ProjectileType` | typedef (L_Enum template) | Projectile type enumeration with numeric and string lookup |
| `Lua_ProjectileTypes` | typedef (L_EnumContainer template) | Projectile type collection supporting mnemonic string lookups |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Lua_Projectile_Name` | extern char[] | global | String constant `"projectile"` identifying the Lua class name |
| `Lua_Projectiles_Name` | extern char[] | global | String constant `"Projectiles"` identifying the container name |
| `Lua_ProjectileType_Name` | extern char[] | global | String constant `"projectile_type"` identifying the enum class name |
| `Lua_ProjectileTypes_Name` | extern char[] | global | String constant `"ProjectileTypes"` identifying the enum container name |

## Key Functions / Methods

### Lua_Projectiles_register
- **Signature:** `int Lua_Projectiles_register(lua_State *L)`
- **Purpose:** Initialize and register all projectile-related Lua bindings with the Lua virtual machine
- **Inputs:** `L` ΓÇô active Lua state
- **Outputs/Return:** Error code (0 on success, non-zero on failure per Lua conventions)
- **Side effects (global state, I/O, alloc):** Modifies Lua state registry; allocates metatable entries and method tables
- **Calls (direct calls visible in this file):** Not visible; implementation is in a separate .cpp file
- **Notes:** Called during engine initialization to expose projectile API to Lua scripts

## Control Flow Notes
This is a header-only declaration file. Runtime initialization occurs when `Lua_Projectiles_register()` is called (likely during Lua state setup). After registration, Lua scripts can create, access, and modify projectile objects and query projectile types.

## External Dependencies
- **Bundled headers:**
  - `cseries.h` ΓÇô project compatibility/foundation layer
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua 5.1 C API (conditional on `HAVE_LUA`)
  - `lua_templates.h` ΓÇô template classes (`L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`) that power all bindings
