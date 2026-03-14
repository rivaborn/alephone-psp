# Source_Files/Lua/lua_projectiles.cpp

## File Purpose
Implements Lua bindings for the projectile system in the Aleph One game engine (Marathon). Exposes projectile properties, creation, and management to Lua scripts through a template-based class wrapping system.

## Core Responsibilities
- Expose projectile properties (position, orientation, damage, owner, target) as Lua accessors
- Convert between game engine units (fixed-point angles, world units) and Lua-friendly formats (degrees, floating-point coordinates)
- Provide Lua methods to create new projectiles and manipulate their state
- Register Lua classes for projectile types and associated damage definitions
- Define backward-compatibility layer for legacy Lua scripts
- Manage validity checks for projectile indices

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `projectile_data` | struct (defined elsewhere) | Core projectile state; accessed via index |
| `object_data` | struct (defined elsewhere) | Shared object representation; holds location, facing, polygon |
| `world_point3d` | struct (defined elsewhere) | 3D position in world coordinates |
| `Lua_Projectile` | typedef L_Class | Lua-wrapped individual projectile instance |
| `Lua_Projectiles` | typedef L_Container | Lua-wrapped collection of all projectiles |
| `Lua_ProjectileType` | typedef L_Enum | Lua-wrapped projectile type identifier |
| `Lua_ProjectileTypeDamage` | typedef L_Class | Lua-wrapped damage info for a projectile type |
| `Lua_ProjectileTypes` | typedef L_EnumContainer | Lua-wrapped collection of all projectile types |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | const float | file-static | Conversion factor: 360 / FULL_CIRCLE; converts engine angles to degrees |
| `Lua_Projectile_Name` | char[] | global | String identifier "projectile" for Lua class registration |
| `Lua_Projectiles_Name` | char[] | global | String identifier "Projectiles" for Lua container |
| `Lua_ProjectileType_Name` | char[] | global | String identifier "projectile_type" for Lua enum |
| `Lua_ProjectileTypeDamage_Name` | char[] | global | String identifier "projectile_type_damage" |
| `Lua_ProjectileTypes_Name` | char[] | global | String identifier "ProjectileTypes" for enum container |
| `compatibility_script` | const char[] | file-static | Embedded Lua code for backward-compatible wrapper functions |

## Key Functions / Methods

### Lua_Projectile_Get_Damage_Scale
- **Signature:** `static int Lua_Projectile_Get_Damage_Scale(lua_State *L)`
- **Purpose:** Lua getter to retrieve projectile damage scaling multiplier
- **Inputs:** Lua state; assumes projectile object at stack index 1
- **Outputs/Return:** Pushes damage_scale (normalized to 1.0 = FIXED_ONE) to Lua stack; returns 1
- **Side effects:** Reads projectile_data from global projectiles array
- **Calls:** `get_projectile_data()`, `lua_pushnumber()`

### Lua_Projectile_Get_Elevation
- **Signature:** `static int Lua_Projectile_Get_Elevation(lua_State *L)`
- **Purpose:** Lua getter for projectile pitch/elevation angle
- **Inputs:** Lua state; projectile index at stack 1
- **Outputs/Return:** Pushes elevation in degrees (-180 to 180) to stack; returns 1
- **Side effects:** Converts engine angle units to degrees; normalizes to [-180, 180) range
- **Calls:** `get_projectile_data()`, `lua_pushnumber()`
- **Notes:** Also accessible via "pitch" alias in getter table

### Lua_Projectile_Get_Facing
- **Signature:** `static int Lua_Projectile_Get_Facing(lua_State *L)`
- **Purpose:** Lua getter for projectile yaw (heading)
- **Inputs:** Lua state; projectile index at stack 1
- **Outputs/Return:** Pushes facing angle in degrees to stack; returns 1
- **Side effects:** Reads associated object_data via projectileΓåÆobject_index
- **Calls:** `get_projectile_data()`, `get_object_data()`, `lua_pushnumber()`
- **Notes:** Also accessible via "yaw" alias

### Lua_Projectile_Get_Gravity / Lua_Projectile_Get_Owner / Lua_Projectile_Get_Polygon / Lua_Projectile_Get_Target / Lua_Projectile_Get_Type / Lua_Projectile_Get_X/Y/Z
- **Pattern:** Similar getter structure; retrieve specific projectile properties
- **Gravity:** Returns dz (vertical acceleration) normalized to world units
- **Owner/Target:** Return associated monster index or nil
- **Polygon:** Returns polygon containment index
- **Type:** Returns projectile type enum
- **X/Y/Z:** Return world coordinates normalized to double precision

### Lua_Projectile_Set_* (setters)
- **Pattern:** Validate Lua argument type, retrieve projectile and/or object data, store modified value back
- **Lua_Projectile_Set_Damage_Scale, Set_Elevation, Set_Facing, Set_Gravity:** Perform unit conversions (degreesΓåÆengine angles, floatsΓåÆfixed-point)
- **Lua_Projectile_Set_Owner / Set_Target:** Accept nil, number, Lua_Monster, or Lua_Player; resolve to monster index
- **Notes:** Setters return 0 (no Lua return value); error via luaL_error on type mismatch

### Lua_Projectile_Play_Sound
- **Signature:** `int Lua_Projectile_Play_Sound(lua_State *L)`
- **Purpose:** Play a sound effect at the projectile's location
- **Inputs:** Projectile index (stack 1), sound code (stack 2)
- **Outputs/Return:** Returns 0
- **Side effects:** Calls play_object_sound() to emit sound
- **Calls:** `Lua_Sound::ToIndex()`, `get_projectile_data()`, `play_object_sound()`

### Lua_Projectile_Position
- **Signature:** `int Lua_Projectile_Position(lua_State *L)`
- **Purpose:** Reposition projectile to new location and optionally new polygon
- **Inputs:** Projectile index (1), x/y/z coordinates (2ΓÇô4), polygon index or object (5)
- **Outputs/Return:** Returns 0
- **Side effects:** Updates object_data location; if polygon changes, updates polygon object lists
- **Calls:** `get_projectile_data()`, `get_object_data()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()`
- **Notes:** Validates polygon index; converts float coordinates to fixed-point (multiply by WORLD_ONE)

### Lua_Projectiles_New_Projectile
- **Signature:** `static int Lua_Projectiles_New_Projectile(lua_State *L)`
- **Purpose:** Create and spawn a new projectile at a specified location
- **Inputs:** x/y/z position (1ΓÇô3), polygon index or object (4), projectile type (5)
- **Outputs/Return:** Pushes new projectile object to Lua stack; returns 1
- **Side effects:** Allocates projectile slot; initializes with default velocity vector (1,0,0) and damage scale FIXED_ONE
- **Calls:** `Lua_Polygon::Valid()`, `Lua_Polygon::Index()`, `Lua_ProjectileType::ToIndex()`, `::new_projectile()`, `Lua_Projectile::Push()`
- **Notes:** Hardcoded initial velocity; owner/target initialized to NONE

### Lua_Projectile_Valid
- **Signature:** `bool Lua_Projectile_Valid(int32 index)`
- **Purpose:** Check if a projectile index is in bounds and slot is in use
- **Inputs:** Projectile index
- **Outputs/Return:** true if valid and used; false otherwise
- **Calls:** `GetMemberWithBounds()`, `SLOT_IS_USED()` macro
- **Notes:** Called during Lua class Push/access; bounds check against MAXIMUM_PROJECTILES_PER_MAP

### Lua_Projectiles_register
- **Signature:** `int Lua_Projectiles_register(lua_State *L)`
- **Purpose:** Register all projectile-related Lua classes and containers with the Lua state
- **Inputs:** Lua state
- **Outputs/Return:** Returns 0
- **Side effects:** Registers Lua_Projectile, Lua_Projectiles, Lua_ProjectileType, Lua_ProjectileTypeDamage, and Lua_ProjectileTypes; sets up getter/setter tables; binds length callbacks; loads compatibility script
- **Calls:** `L_Class::Register()`, `L_Enum::Register()`, `L_Container::Register()`, `boost::bind()`, `get_dynamic_limit()`, `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** Projectile length is bound to dynamic limit for _dynamic_limit_projectiles; projectile type length is constant (NUMBER_OF_PROJECTILE_TYPES)

### compatibility() [static]
- **Signature:** `static void compatibility(lua_State *L)`
- **Purpose:** Load legacy Lua functions for scripts written against older engine API
- **Inputs:** Lua state
- **Side effects:** Parses and executes embedded Lua script defining wrapper functions
- **Calls:** `luaL_loadbuffer()`, `lua_pcall()`
- **Notes:** Defines functions like get_projectile_angle(), set_projectile_position() that map to new class-based API

## Control Flow Notes
This file is a pure Lua binding layer; it does not participate in core game loop. Initialization occurs in `Lua_Projectiles_register()`, called during Lua VM setup. Property access (getters/setters) occurs on-demand when Lua scripts access projectile fields. The compatibility script runs once at registration to backfill old API.

## External Dependencies
- **Notable includes:**
  - `lua.h`, `lauxlib.h`, `lualib.h` (Lua C API)
  - `lua_templates.h` (template-based Lua binding infrastructure: L_Class, L_Container, L_Enum)
  - `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h` (other Lua bindings)
  - `dynamic_limits.h` (get_dynamic_limit)
  - `map.h`, `monsters.h`, `player.h`, `projectiles.h` (core engine data structures and functions)
  - `boost/bind.hpp` (boost::bind for callback binding)

- **Defined elsewhere:**
  - `get_projectile_data()`, `::new_projectile()` ΓÇö projectile management
  - `get_object_data()`, `play_object_sound()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()` ΓÇö object/polygon management
  - `Lua_Monster::Push()`, `Lua_Player::Is()`, `Lua_Polygon::Push()`, `Lua_Sound::ToIndex()`, `Lua_ProjectileType::ToIndex()`, `Lua_DamageType::Push()` ΓÇö other Lua class bindings
  - `SLOT_IS_USED()` macro ΓÇö slot validity check
  - `WORLD_ONE`, `FIXED_ONE`, `FULL_CIRCLE` constants ΓÇö unit conversion factors
  - `projectile_definitions.h` (included with DONT_REPEAT_DEFINITIONS macro) ΓÇö projectile type definitions
