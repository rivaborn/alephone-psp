# Source_Files/Lua/lua_templates.h

## File Purpose

Defines C++ template classes and functions for exposing game engine objects and containers to Lua scripts. This is the core Lua/C++ binding infrastructureΓÇöit wraps indexed game entities (monsters, items, players) and enumerations as Lua objects, allowing Lua scripts to query and manipulate them with automatic type checking and method dispatch.

## Core Responsibilities

- Provide template-based class registration (`L_Class`) to expose indexed objects to Lua
- Implement object wrapping and instance caching to maintain object identity across Lua calls
- Define validation and access predicates (`Valid`, `Length`) as pluggable callbacks
- Support enumeration types with mnemonic name-to-value mapping (`L_Enum`)
- Implement container-like access patterns (array indexing, iteration, method calls) via `L_Container`
- Manage Lua registry storage for method tables, instance caches, and mnemonic tables
- Provide metamethod handlers (`__index`, `__newindex`, `__tostring`, `__eq`) for Lua objects
- Support string-based (mnemonic) lookup for enum containers (`L_EnumContainer`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| L_TableFunction | function template | Wraps a Lua C function and pushes it onto the stack |
| L_Class | class template | Base wrapper for indexed objects with generic getter/setter support |
| ValidRange | struct | Predicate that checks if an index falls within a range [0, max_index) |
| always_valid | struct | Predicate that always returns true (no validation) |
| L_Enum | class template | Extends L_Class with equality testing and mnemonic name-to-value mapping |
| L_LazyEnum | class template | L_Enum variant with deferred error handling for invalid indices |
| L_Container | class template | Provides table-like indexed access to a collection with iteration support |
| ConstantLength | struct | Predicate returning a fixed container length |
| L_EnumContainer | class template | Extends L_Container to support string-based (mnemonic) key lookups |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Valid | boost::function<bool(index_t)> | static member of L_Class | Validation predicate; checks if an index is valid before accessing the object |
| Length | boost::function<typename T::index_type()> | static member of L_Container | Returns the current length/size of the container |

## Key Functions / Methods

### L_Class<name, index_t>::Register
- **Signature:** `static void Register(lua_State *L, const luaL_reg get[] = 0, const luaL_reg set[] = 0, const luaL_reg metatable[] = 0)`
- **Purpose:** Register a Lua class by creating its metatable, method tables, and instance cache in the registry
- **Inputs:** Lua state, optional getter methods, setter methods, and custom metamethods
- **Outputs/Return:** None
- **Side effects:** Modifies Lua registry; creates metatables and global function (e.g., `is_<classname>`)
- **Calls:** luaL_newmetatable, lua_pushcfunction, lua_settable, luaL_openlib, lua_setglobal
- **Notes:** Stores method tables under light userdata keys derived from the `name` pointer (e.g., `&name[1]` for getters, `&name[2]` for setters, `&name[3]` for instances, `&name[4]` for mnemonics)

### L_Class<name, index_t>::Push
- **Signature:** `static L_Class<name, index_t> *Push(lua_State *L, index_t index)`
- **Purpose:** Push a Lua object wrapper onto the stack for a game engine object at the given index
- **Inputs:** Lua state, native index (e.g., monster index, item index)
- **Outputs/Return:** Pointer to the L_Class wrapper; nil on stack if index is invalid
- **Side effects:** Allocates Lua userdata; may create/retrieve from instance cache
- **Calls:** Valid(), lua_gettable, lua_newtable, lua_newuserdata, luaL_getmetatable, lua_setmetatable
- **Notes:** Implements instance cachingΓÇöreuses existing Lua wrapper if already created; validates index via Valid() predicate

### L_Class<name, index_t>::Index
- **Signature:** `static index_t Index(lua_State *L, int index)`
- **Purpose:** Extract the native index from a Lua userdata object
- **Inputs:** Lua state, stack index of the userdata
- **Outputs/Return:** Native index value
- **Side effects:** Raises Lua type error if object is not the expected type
- **Calls:** lua_touserdata, luaL_typerror
- **Notes:** Performs type checking; used internally by accessor methods

### L_Class<name, index_t>::Is
- **Signature:** `static bool Is(lua_State *L, int index)`
- **Purpose:** Check if a Lua value is an instance of this class (runtime type check)
- **Inputs:** Lua state, stack index
- **Outputs/Return:** Boolean (true if the userdata's metatable matches this class's metatable)
- **Side effects:** None
- **Calls:** lua_touserdata, lua_getmetatable, lua_getfield, lua_rawequal, lua_pop
- **Notes:** Safely handles non-userdata values by returning false

### L_Class<name, index_t>::Invalidate
- **Signature:** `static void Invalidate(lua_State *L, index_t index)`
- **Purpose:** Remove a stale object from the instance cache (e.g., when the underlying entity is destroyed)
- **Inputs:** Lua state, native index to invalidate
- **Outputs/Return:** None
- **Side effects:** Removes entry from the instance table in the registry
- **Calls:** lua_gettable, lua_pushnil, lua_settable, lua_pop
- **Notes:** Does not destroy the Lua userdata; existing references become invalid

### L_Enum<name, index_t>::Register
- **Signature:** `static void Register(lua_State *L, const luaL_reg get[], const luaL_reg set[], const luaL_reg metatable[], const lang_def mnemonics[] = 0)`
- **Purpose:** Register an enumeration class with mnemonic string-to-value mapping
- **Inputs:** Lua state, getter/setter methods, metatable methods, mnemonic array (name-value pairs)
- **Outputs/Return:** None
- **Side effects:** Calls L_Class::Register; adds `__eq` metamethod for numeric comparison; creates mnemonic table in registry; replaces `__tostring` with mnemonic lookup
- **Calls:** L_Class<>::Register, lua_pushstring, lua_pushcfunction, lua_settable, luaL_openlib
- **Notes:** Mnemonics are stored bidirectionally (nameΓåÆvalue and valueΓåÆname) for efficient lookup in both directions

### L_Enum<name, index_t>::ToIndex
- **Signature:** `static index_t ToIndex(lua_State *L, int index)`
- **Purpose:** Convert a Lua value (number, string mnemonic, or userdata) to a native index
- **Inputs:** Lua state, stack index of the value to convert
- **Outputs/Return:** Native index; raises Lua error if conversion fails or index is invalid
- **Side effects:** Performs mnemonic lookup in registry; raises Lua errors for invalid/incorrect argument types
- **Calls:** _lookup, luaL_error
- **Notes:** Supports numeric lookup (e.g., `0` ΓåÆ index 0), string mnemonic lookup (e.g., `"pistol"` ΓåÆ index 1), and userdata passthrough

### L_Container<name, T>::Register
- **Signature:** `static void Register(lua_State *L, const luaL_reg methods[] = 0, const luaL_reg metatable[] = 0)`
- **Purpose:** Create a global table-like object for indexed container access (e.g., a "monsters" or "items" collection)
- **Inputs:** Lua state, optional container methods and metamethods
- **Outputs/Return:** None
- **Side effects:** Creates global variable named `name`; creates metatable; stores methods in registry
- **Calls:** lua_newuserdata, luaL_newmetatable, lua_pushcfunction, lua_settable, lua_setglobal
- **Notes:** The global is a userdata with a metatable, not a regular table; supports both numeric indexing and method calls

## Control Flow Notes

**Initialization/Registration:**
- `Register()` is called once at game startup to expose a class/container to Lua
- Sets up metatables, method tables, and registry storage

**Object access (per-frame or on-demand):**
1. Lua script requests an object by index or container method
2. `Push()` creates or retrieves a cached Lua wrapper userdata
3. When Lua accesses a property (e.g., `monster.health`), `__index` metamethod calls `_get()`
4. `_get()` looks up the property name in the registered getter methods table and invokes the corresponding C function

**Invalidation:**
- When a game engine object is destroyed, `Invalidate()` is called to remove it from the cache
- Existing Lua references become invalid but do not crash immediately

**Enum/mnemonic lookup:**
- `ToIndex()` and `_lookup()` resolve string names to numeric indices by querying the mnemonic table
- Support both direct numeric access and string-based access

## External Dependencies

- **Lua C API:** lua.h, lauxlib.h, lualib.h ΓÇö Core Lua embedding and auxiliary library functions (luaL_newmetatable, lua_pushcfunction, lua_pcall, etc.)
- **Boost:** boost/function.hpp ΓÇö Function wrapper for pluggable validation/length predicates
- **Game engine:** lua_script.h (game event callbacks), lua_mnemonics.h (lang_def mnemonic arrays), cseries.h (engine types and macros)
- **Standard library:** sstream (std::ostringstream for formatting __tostring output)
