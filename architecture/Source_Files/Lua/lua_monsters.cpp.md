# Source_Files/Lua/lua_monsters.cpp

## File Purpose
Implements Lua scripting bindings for monsters in the game engine. Allows Lua scripts to create monsters, query/modify their properties (position, health, mode, action), and trigger actions (damage, pathfinding, sound effects).

## Core Responsibilities
- Register Lua monster classes (`Lua_Monster`, `Lua_MonsterType`, `Lua_MonsterClass`) with the Lua state
- Expose monster instance methods (attack, damage, move_by_path, play_sound, position)
- Provide property accessors for monster types (class, enemies, friends, immunities, weaknesses, etc.)
- Provide property accessors for monster instances (vitality, facing, mode, action, polygon, visibility)
- Implement factory method for creating new monsters
- Provide backward-compatibility layer of wrapper functions for old Lua API

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Lua_Monster | L_Class template | Lua class representing a single monster instance |
| Lua_MonsterType | L_Enum template | Lua enum representing monster type definitions |
| Lua_MonsterClass | L_Enum template | Lua enum for monster class bitmasks (e.g., yeti, fighter) |
| Lua_MonsterMode | L_Enum template | Lua enum for monster AI mode (locked, lost_lock, unlocked, running) |
| Lua_MonsterAction | L_Enum template | Lua enum for monster animation state (stationary, moving, attacking, dying) |
| Lua_MonsterType_Enemies | L_Class template | Property table for monster type's enemy relationships |
| Lua_MonsterType_Friends | L_Class template | Property table for monster type's friendly relationships |
| Lua_MonsterType_Immunities | L_Class template | Property table for damage type immunities |
| Lua_MonsterType_Weaknesses | L_Class template | Property table for damage type weaknesses |
| monster_pathfinding_data | struct | Temporary data passed to pathfinding cost function |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| AngleConvert | const float | file-static | Constant for converting internal angle units (FULL_CIRCLE) to degrees (360┬░) |
| Lua_Monster_Name | char[] | extern | String identifier "monster" for Lua class registration |
| Lua_MonsterType_Name | char[] | static | String identifier "monster_type" for Lua enum registration |
| Lua_MonsterClass_Name | char[] | static | String identifier "monster_class" for Lua enum registration |
| Various *_Metatable arrays | luaL_reg[] | static | Lua method tables for __index and __newindex metamethods |

## Key Functions / Methods

### Lua_Monsters_register
- Signature: `int Lua_Monsters_register(lua_State *L)`
- Purpose: Register all monster-related Lua classes, enums, and methods with the Lua state during engine initialization
- Inputs: Lua state pointer
- Outputs/Return: 0 (success)
- Side effects: Modifies Lua registry to add Monsters table and all monster classes/enums
- Calls: Lua_MonsterClass::Register, Lua_MonsterType::Register, Lua_Monster::Register, Lua_Monsters::Register, compatibility()
- Notes: Called once at startup; sets up the entire Lua monster API

### Lua_Monsters_New
- Signature: `int Lua_Monsters_New(lua_State *L)`
- Purpose: Factory method called as `Monsters.new(x, y, z, polygon, type)` to create new monster instances
- Inputs: Stack contains x, y, z coordinates (world units), polygon index, and monster type
- Outputs/Return: Pushes new Lua_Monster object onto stack; returns 1
- Side effects: Creates new monster_data in engine, allocates resources
- Calls: ::new_monster(), Lua_Monster::Push()
- Notes: Returns 0 (nil) if monster creation fails (e.g., max monsters reached)

### Lua_Monster_Attack
- Signature: `int Lua_Monster_Attack(lua_State *L)`
- Purpose: Set a monster's attack target by index
- Inputs: Stack[1] = monster instance, Stack[2] = target (monster index or Lua_Monster object)
- Outputs/Return: 0
- Side effects: Modifies monster AI target
- Calls: change_monster_target()

### Lua_Monster_Damage
- Signature: `int Lua_Monster_Damage(lua_State *L)`
- Purpose: Apply damage to a monster
- Inputs: Stack[1] = monster, Stack[2] = damage amount, Stack[3] (optional) = damage type enum
- Outputs/Return: 0
- Side effects: Modifies monster vitality, may trigger death animation
- Calls: damage_monster()
- Notes: If no damage type specified, defaults to _damage_fist

### Lua_Monster_Move_By_Path
- Signature: `int Lua_Monster_Move_By_Path(lua_State *L)`
- Purpose: Command monster to pathfind and move to a destination polygon
- Inputs: Stack[1] = monster, Stack[2] = destination polygon (index or Lua_Polygon)
- Outputs/Return: 0
- Side effects: Activates monster if inactive, allocates/updates pathfinding path, sets monster movement goal
- Calls: get_monster_data(), get_monster_definition_external(), get_object_data(), activate_monster(), delete_path(), new_path(), set_monster_action(), set_monster_mode(), advance_monster_path()
- Notes: Sets monster to unlocked mode if path cannot be computed

### Lua_Monster_Position
- Signature: `int Lua_Monster_Position(lua_State *L)`
- Purpose: Set monster's position and polygon
- Inputs: Stack[1] = monster, Stack[2ΓÇô4] = x, y, z (in world units), Stack[5] = polygon index
- Outputs/Return: 0
- Side effects: Moves monster object in game world, updates polygon membership
- Calls: get_monster_data(), get_object_data(), remove_object_from_polygon_object_list(), add_object_to_polygon_object_list()

### Lua_Monster_Play_Sound
- Signature: `int Lua_Monster_Play_Sound(lua_State *L)`
- Purpose: Play a sound at monster's location
- Inputs: Stack[1] = monster, Stack[2] = sound type (Lua_Sound enum)
- Outputs/Return: 0
- Side effects: Plays 3D sound in game world
- Calls: get_monster_data(), get_object_data(), play_object_sound()

### Lua_MonsterType_Enemies_{Get,Set}
- Signature: `static int Lua_MonsterType_Enemies_{Get,Set}(lua_State *L)`
- Purpose: Get/set enemy relationship flags for a monster type
- Inputs: Stack[1] = monster type, Stack[2] = enemy class enum, Stack[3] (Set only) = boolean
- Outputs/Return: Get returns 1 (boolean); Set returns 0
- Side effects: Modifies monster_definition in engine state
- Calls: get_monster_definition_external()
- Notes: Uses bitwise OR/AND to set/clear individual class bits

### Monster Property Getters (static, ~15 functions)
Examples: `Lua_Monster_Get_X`, `Lua_Monster_Get_Y`, `Lua_Monster_Get_Z`, `Lua_Monster_Get_Vitality`, `Lua_Monster_Get_Visible`, `Lua_Monster_Get_Type`, `Lua_Monster_Get_Polygon`, `Lua_Monster_Get_Mode`, `Lua_Monster_Get_Action`, `Lua_Monster_Get_Active`, `Lua_Monster_Get_Facing`
- Purpose: Read-only accessors for monster instance properties
- Inputs: Stack[1] = monster instance
- Outputs/Return: Pushes property value onto stack; returns 1
- Side effects: None (read-only)
- Calls: get_monster_data(), get_object_data(), Lua_MonsterType::Push(), Lua_MonsterMode::Push(), Lua_MonsterAction::Push()
- Notes: Convert internal units (e.g., WORLD_ONE, radians) to Lua-friendly values (degrees)

### Monster Property Setters (static, ~4 functions)
Examples: `Lua_Monster_Set_Active`, `Lua_Monster_Set_Facing`, `Lua_Monster_Set_Visible`, `Lua_Monster_Set_Vitality`
- Purpose: Write accessors for monster instance properties
- Inputs: Stack[1] = monster, Stack[2] = new property value
- Outputs/Return: 0
- Side effects: Modifies monster_data or object_data in engine
- Calls: get_monster_data(), get_object_data(), activate_monster(), deactivate_monster(), etc.
- Notes: Validate input types before modifying; some setters have restrictions (e.g., cannot set visible on active monsters)

### Helper Functions (static, inline)
- `powerOfTwo()`: Check if integer is a power of 2 (used for monster class validation)
- `Lua_MonsterClass_Valid()`: Validate monster class enum value is in range and power-of-two

## Control Flow Notes
**Initialization phase:** Engine calls `Lua_Monsters_register()` once to set up Lua monster API and load compatibility wrapper functions.

**Runtime/frame update:** Lua scripts access Monsters table (`Monsters[index]` or `Monsters.new(...)`) via registered getters/setters. These marshal data between Lua and internal `monster_data`/`monster_definition` structures. Properties like position, vitality, and AI state are updated each frame by the engine and visible to Lua through the getter interface.

**Backward compatibility:** The `compatibility_script` string at EOF provides old-API function wrappers (e.g., `activate_monster(m)`, `damage_monster(m, dmg)`) by mapping them to the new OOP-style API (`Monsters[m].active = true`, `Monsters[m]:damage(dmg)`).

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (luaL_reg, lua_State, luaL_error, etc.)
- **Template classes:** lua_templates.h (L_Class, L_Enum, L_Container, L_EnumContainer)
- **Engine internals:** 
  - monsters.h (monster_data, monster_definition, MAXIMUM_MONSTERS_PER_MAP, activation/movement/damage functions)
  - flood_map.h (pathfinding: new_path, delete_path, move_along_path)
  - player.h (monster_index_to_player_index)
  - lua_map.h (Lua_Polygon, Lua_DamageType, etc.)
  - lua_objects.h (Lua_EffectType, Lua_ItemType, Lua_Sound)
  - lua_player.h (Lua_Player)
  - boost/bind.hpp (boost::bind for Length callback)
- **External symbols used:** get_monster_data, get_monster_definition_external, get_object_data, change_monster_target, activate_monster, deactivate_monster, damage_monster, ::new_monster, play_object_sound, advance_monster_path, set_monster_action, set_monster_mode, monster_pathfinding_cost_function, get_polygon_data, GetMemberWithBounds, remove_object_from_polygon_object_list, add_object_to_polygon_object_list, monster_index_to_player_index
