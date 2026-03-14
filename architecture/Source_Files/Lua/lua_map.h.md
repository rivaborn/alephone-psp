# Source_Files/Lua/lua_map.h

## File Purpose
Declares Lua-C bindings for map geometry and related elements in the Aleph One engine. Exposes polygons, lines, sides, platforms, lights, tags, terminals, and media to Lua scripts via template-based wrapper classes.

## Core Responsibilities
- Declare Lua-C wrapper typedefs for map structural elements (lines, polygons, sides)
- Declare enum wrapper classes for map-related enumerations (damage types, transfer modes, control panel types, etc.)
- Declare container classes for iterating collections of map elements
- Export the registration function to bind map API to a Lua interpreter

## Key Types / Data Structures

| Name | Kind (struct/enum/class/typedef/interface/trait) | Purpose |
|------|------|---------|
| Lua_Line | typedef via L_Class | Wraps individual map line for Lua access |
| Lua_Lines | typedef via L_Container | Container for iterating all map lines |
| Lua_Polygon | typedef via L_Class | Wraps polygon geometry for Lua access |
| Lua_Polygons | typedef via L_Container | Container for iterating all polygons |
| Lua_Polygon_Ceiling, Lua_Polygon_Floor | typedef via L_Class | Wraps polygon surface (ceiling/floor) data |
| Lua_Platform, Lua_Platforms | typedef via L_Class / L_Container | Wraps moving platform and container |
| Lua_Side, Lua_Sides | typedef via L_Class / L_Container | Wraps wall side and container |
| Lua_Light, Lua_Lights | typedef via L_Class / L_Container | Wraps light source and container |
| Lua_Tag, Lua_Tags | typedef via L_Class / L_Container | Wraps activation tag and container |
| Lua_Terminal, Lua_Terminals | typedef via L_Class / L_Container | Wraps computer terminal and container |
| Lua_Media, Lua_Medias | typedef via L_Class / L_Container | Wraps liquids/media and container |
| Lua_DamageType, Lua_DamageTypes | typedef via L_Enum / L_EnumContainer | Damage type enumeration |
| Lua_ControlPanelType, Lua_ControlPanelTypes | typedef via L_Enum / L_EnumContainer | Control panel type enumeration |
| Lua_SideType, Lua_SideTypes | typedef via L_Enum / L_EnumContainer | Side type enumeration |
| Lua_TransferMode, Lua_TransferModes | typedef via L_Enum / L_EnumContainer | Texture transfer mode enumeration |
| Lua_Collection, Lua_Collections | typedef via L_Enum / L_EnumContainer | Asset collection enumeration |

## Global / File-Static State

| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|---------|---------|
| Lua_Line_Name | extern char[] | global | String identifier "line" for Lua class registration |
| Lua_Polygon_Name | extern char[] | global | String identifier "polygon" for Lua class registration |
| Lua_Platform_Name | extern char[] | global | String identifier "platform" for Lua class registration |
| Lua_Light_Name | extern char[] | global | String identifier "light" for Lua class registration |
| Lua_Tag_Name | extern char[] | global | String identifier "tag" for Lua class registration |
| Lua_Terminal_Name | extern char[] | global | String identifier "terminal" for Lua class registration |
| Lua_Side_Name | extern char[] | global | String identifier "side" for Lua class registration |
| Lua_Media_Name | extern char[] | global | String identifier "media" for Lua class registration |
| Lua_*_Name (for all enums) | extern char[] | global | String identifiers for enum types and containers |

All name variables follow the pattern of declaring extern char arrays used internally by the template system for Lua metatable registration.

## Key Functions / Methods

### Lua_Map_register
- **Signature:** `int Lua_Map_register (lua_State *L)`
- **Purpose:** Register all map-related Lua bindings with the interpreter; called during engine initialization to expose map API to scripts.
- **Inputs:** `L` ΓÇô Lua state to register bindings into
- **Outputs/Return:** Returns integer status code (standard Lua convention); presumably 0 or 1
- **Side effects (global state, I/O, alloc):** Modifies Lua registry to add metatables, methods, and container objects for all map types; allocates Lua tables and userdata wrappers
- **Calls (direct calls visible in this file):** Not visible; implementation in a .cpp file (likely `lua_map.cpp`)
- **Notes (invariants, edge cases):** Header-only declarations; actual registration logic defined elsewhere. Uses template instantiation from `lua_templates.h` to generate bindings for each type.

## Control Flow Notes
This file is initialization-only. `Lua_Map_register()` is invoked during engine startup (likely in a Lua initialization routine) to expose map structures to Lua scripts. It does not participate in frame updates, rendering, or shutdown. Once registered, Lua scripts can access and manipulate map geometry at runtime through the bound API.

## External Dependencies
- **Lua headers:** `lua.h`, `lauxlib.h`, `lualib.h` (Lua 5.1 API)
- **Game headers:** `map.h` (map data structures), `lightsource.h` (light definitions)
- **Template library:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer template definitions)
- **Common headers:** `cseries.h` (cross-platform utilities)
