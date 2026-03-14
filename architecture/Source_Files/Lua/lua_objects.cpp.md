# Source_Files/Lua/lua_objects.cpp

## File Purpose
Implements Lua bindings for game map objects (Effects, Items, Scenery). Exposes object creation, deletion, and property access to Lua scripts via template-based wrappers around C API calls. Provides constructors, getters/setters, and registration with the Lua VM.

## Core Responsibilities
- Register object classes (Effects, Items, Scenery) with Lua metatable system
- Implement object constructors (`Effects.new()`, `Items.new()`, `Scenery.new()`)
- Expose object properties (position, facing, polygon, type) as Lua getter/setter functions
- Handle object deletion with cleanup (e.g., deanimation for scenery)
- Provide object sound playback interface
- Validate object indices and enforce state constraints
- Maintain backwards-compatibility wrapper functions for legacy Lua scripts

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Lua_Effect` | typedef (L_Class) | Lua userdata wrapper for effect objects |
| `Lua_Effects` | typedef (L_Container) | Collection accessor for all effects on map |
| `Lua_Item` | typedef (L_Class) | Lua userdata wrapper for item objects |
| `Lua_Items` | typedef (L_Container) | Collection accessor for all items on map |
| `Lua_Scenery` | typedef (L_Class) | Lua userdata wrapper for scenery objects |
| `Lua_Sceneries` | typedef (L_Container) | Collection accessor for all scenery on map |
| `Lua_EffectType` | typedef (L_Enum) | Enumeration of effect type constants |
| `Lua_ItemType` | typedef (L_Enum) | Enumeration of item type constants |
| `Lua_SceneryType` | typedef (L_Enum) | Enumeration of scenery type constants |
| `Lua_Sound` | typedef (L_LazyEnum) | Lazy-loaded sound definition reference |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `AngleConvert` | const float | file-static | Conversion factor from internal angle units to degrees (360/FULL_CIRCLE) |
| `Lua_*_Name[]` | char[] | extern/file-static | Lua class/enum names for metatable registration |
| `Lua_*_Get[]` | const luaL_reg[] | file-static | Getter method tables for object property access |
| `Lua_*_Set[]` | const luaL_reg[] | file-static | Setter method tables for object property modification |
| `Lua_*_Methods[]` | const luaL_reg[] | file-static | Constructor and utility method tables |
| `compatibility_script` | const char* | file-static | Legacy Lua functions for backwards compatibility |

## Key Functions / Methods

### lua_delete_object<T> (template)
- Signature: `template<class T> int lua_delete_object(lua_State *L)`
- Purpose: Remove a map object by index
- Inputs: Lua state, object index on stack[1]
- Outputs/Return: 0 (no return value to Lua)
- Side effects: Calls `remove_map_object()`
- Calls: `T::Index()`, `remove_map_object()`
- Notes: Specialization for `Lua_Scenery` also calls `deanimate_scenery()` to stop animation loops

### lua_delete_object<Lua_Scenery> (specialization)
- Signature: `template<> int lua_delete_object<Lua_Scenery>(lua_State *L)`
- Purpose: Remove scenery with cleanup
- Inputs: Lua state, scenery index on stack[1]
- Outputs/Return: 0
- Side effects: Removes object and stops animation
- Calls: `remove_map_object()`, `deanimate_scenery()`

### lua_play_object_sound<T> (template)
- Signature: `template<class T> int lua_play_object_sound(lua_State *L)`
- Purpose: Play a sound effect at object location
- Inputs: Lua state, object index [1], sound code [2]
- Outputs/Return: 0
- Side effects: Triggers sound playback
- Calls: `Lua_Sound::ToIndex()`, `play_object_sound()`

### get_object_facing<T> / set_object_facing<T>
- Signature: `template<class T> static int get_object_facing(lua_State *L)`
- Purpose: Query or set object facing angle
- Inputs: Lua state, object index [1], (angle for setter) [2]
- Outputs/Return: 1 (number on stack for getter); 0 for setter
- Side effects: Modifies object->facing via pointer (setter only)
- Calls: `get_object_data()`, angle conversion via `AngleConvert`
- Notes: Converts between internal units and degrees

### get_object_x / get_object_y / get_object_z<T>
- Signature: `template<class T> static int get_object_[xyz](lua_State *L)`
- Purpose: Query object world coordinates
- Inputs: Lua state, object index [1]
- Outputs/Return: 1 (number pushed)
- Calls: `get_object_data()`
- Notes: Divides by `WORLD_ONE` to convert from internal coordinates

### Lua_Effects_New / Lua_Items_New / Lua_Sceneries_New
- Signature: `static int Lua_Effects_New(lua_State *L)` (and similar)
- Purpose: Create new object at specified location
- Inputs: x [1], y [2], height [3], polygon [4] (int or Lua_Polygon), type [5]
- Outputs/Return: 1 (pushes created object or nil if failed)
- Side effects: Allocates object slot, may modify dynamic world state
- Calls: `Lua_Polygon::Valid()`, `Lua_Polygon::Index()`, `::new_effect()` / `::new_item()` / `::new_scenery()`
- Notes: Polygon param accepts both numeric index and Lua object; Scenery specialization also calls `randomize_scenery_shape()`

### Lua_Scenery_Get_Damaged / Lua_Scenery_Get_Solid
- Signature: `static int Lua_Scenery_Get_Damaged(lua_State *L)`
- Purpose: Query if scenery is damaged or solid
- Inputs: Lua state, scenery index [1]
- Outputs/Return: 1 (boolean)
- Calls: `get_object_data()`, macro checks (`GET_OBJECT_OWNER()`, `OBJECT_IS_SOLID()`)

### Lua_Scenery_Damage
- Signature: `int Lua_Scenery_Damage(lua_State *L)`
- Purpose: Damage scenery object
- Inputs: Lua state, scenery index [1]
- Outputs/Return: 0
- Side effects: Modifies scenery state
- Calls: `damage_scenery()`

### Lua_Scenery_Set_Solid
- Signature: `static int Lua_Scenery_Set_Solid(lua_State *L)`
- Purpose: Toggle scenery solidity
- Inputs: Lua state, scenery index [1], boolean [2]
- Outputs/Return: 0
- Side effects: Modifies collision state via `SET_OBJECT_SOLIDITY()`

### Lua_Effect_Valid / Lua_Item_Valid / Lua_Scenery_Valid
- Signature: `bool Lua_Effect_Valid(int32 index)`
- Purpose: Check if object index points to valid object of type
- Inputs: Object index
- Outputs/Return: true if valid
- Calls: `GetMemberWithBounds()`, macro checks (`SLOT_IS_USED()`, `GET_OBJECT_OWNER()`)
- Notes: `Lua_Scenery_Valid` has complex logic to distinguish scenery from player body parts

### Lua_Objects_register
- Signature: `int Lua_Objects_register(lua_State *L)`
- Purpose: Register all object classes with Lua VM at initialization
- Inputs: Lua state
- Outputs/Return: 0
- Side effects: Sets up metatables, enumerations, container accessors in Lua globals
- Calls: `Lua_*::Register()`, `compatibility()`, boost::bind for length callbacks
- Notes: Assigns valid() function pointers and length callbacks for dynamic limits

### compatibility (static)
- Signature: `static void compatibility(lua_State *L)`
- Purpose: Load legacy Lua wrapper functions for backwards compatibility
- Inputs: Lua state
- Outputs/Return: void
- Side effects: Executes Lua code to define wrapper functions
- Notes: Implements functions like `delete_item()`, `new_item()`, `item_index_valid()` that delegate to new API

## Control Flow Notes
**Initialization Phase**: `Lua_Objects_register()` is called during engine startup to populate the Lua registry with object class metadata, method tables, and validation callbacks.

**Object Creation Flow**: `Lua_*_New()` functions validate input parameters (polygon index, type), construct `object_location` or `world_point3d` struct, call engine allocation functions (`::new_effect/item/scenery`), and push resulting object to Lua stack.

**Object Access Flow**: Lua userdata access routes through `L_Class` metatable `__index`/`__newindex` ΓåÆ template methods ΓåÆ `get_object_data()` + property reads/writes.

**Cleanup**: Special handling for scenery ensures animation state is cleared on deletion via `deanimate_scenery()`.

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h`
- **Game Engine Objects**: `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h` (for object_data access)
- **Map/World**: `map.h` (from lua_map.h), `scenery_definitions.h` (for scenery type mnemonics)
- **Templates/Bindings**: `lua_templates.h` (L_Class, L_Container, L_Enum base classes), `lua_map.h` (Lua_Polygon class)
- **Sound**: `SoundManager.h`
- **Boost**: `boost/bind.hpp` (for dynamic limit callbacks)
- **Defined Elsewhere**: `new_effect()`, `new_item()`, `new_scenery()`, `remove_map_object()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`, `play_object_sound()`, `get_dynamic_limit()`, object owner enum constants
