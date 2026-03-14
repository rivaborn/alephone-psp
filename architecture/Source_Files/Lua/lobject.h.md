# Source_Files/Lua/lobject.h

## File Purpose
Defines the core type system for Lua objects, including the tagged value representation (`TValue`) that holds all Lua values, and data structures for collectable objects (strings, tables, functions, closures, upvalues). This is the foundation for type representation throughout the Lua runtime.

## Core Responsibilities
- Define the union-based tagged value system (`Value` + type tag) for representing all Lua types
- Provide macros for runtime type checking and safe value access
- Define structures for all collectable garbage-collected objects (strings, tables, functions, closures, upvalues, userdata)
- Define function prototypes (`Proto`) with metadata for code, constants, local variables, and upvalues
- Define hash table implementation (`Table`, `Node`, `TKey`) with array and hash components
- Export utility functions for type conversion and object comparison

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Value` | union | Holds the actual value: GC pointer, void pointer, number, or boolean |
| `TValue` | struct | Tagged value (Value + type tag); fundamental unit for all Lua values |
| `GCObject` | union | Base type for all collectable objects (forward declaration) |
| `GCheader` | struct | Common header for GC objects (next pointer, type tag, mark byte) |
| `TString` | union | String object with alignment, reserved byte, hash, length |
| `Udata` | union | Userdata object with metatable, environment, and size |
| `Proto` | struct | Function prototype: constants, code, nested functions, line info, locals, upvalues |
| `LocVar` | struct | Local variable metadata (name, PC range where active) |
| `UpVal` | struct | Upvalue: points to stack or owns value; linked list when open |
| `CClosure` | struct | C function closure (function pointer + upvalues array) |
| `LClosure` | struct | Lua function closure (prototype + upvalues array) |
| `Closure` | union | Union of C and Lua closures |
| `Table` | struct | Hash table: flags, metatable, array part, hash part with lastfree pointer |
| `TKey` | union | Hash key (tagged value + chain pointer) |
| `Node` | struct | Hash table node (value + key) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luaO_nilobject_` | `TValue` | global | Constant nil object used as a sentinel |

## Key Functions / Methods

### luaO_log2
- Signature: `int luaO_log2(unsigned int x)`
- Purpose: Compute ceiling of logΓéé(x)
- Inputs: Unsigned integer x
- Outputs/Return: LogΓéé(x) rounded up
- Side effects: None
- Calls: None (defined elsewhere)
- Notes: Used for hash table sizing

### luaO_int2fb / luaO_fb2int
- Signature: `int luaO_int2fb(unsigned int x)` / `int luaO_fb2int(int x)`
- Purpose: Convert between integer and floating-point byte representations
- Inputs: Integer or FB-encoded value
- Outputs/Return: Converted value
- Side effects: None
- Calls: None (defined elsewhere)
- Notes: FB encoding is a space-efficient format for sizes

### luaO_rawequalObj
- Signature: `int luaO_rawequalObj(const TValue *t1, const TValue *t2)`
- Purpose: Compare two TValues for raw equality (without metamethods)
- Inputs: Two TValue pointers
- Outputs/Return: 1 if equal, 0 otherwise
- Side effects: None
- Calls: None (defined elsewhere)
- Notes: Used by raw equality comparisons in VM

### luaO_str2d
- Signature: `int luaO_str2d(const char *s, lua_Number *result)`
- Purpose: Parse a string as a Lua number
- Inputs: String, result pointer
- Outputs/Return: Success flag; stores parsed number in result
- Side effects: Writes to result pointer
- Calls: None (defined elsewhere)
- Notes: Handles Lua's number syntax

### luaO_pushvfstring / luaO_pushfstring
- Signature: `const char *luaO_pushvfstring(lua_State *L, const char *fmt, va_list argp)` / `const char *luaO_pushfstring(lua_State *L, const char *fmt, ...)`
- Purpose: Format and push a string onto the Lua stack (variadic/va_list versions)
- Inputs: Lua state, format string, arguments
- Outputs/Return: Pointer to the pushed string
- Side effects: Modifies stack, allocates string
- Calls: None (defined elsewhere)
- Notes: Similar to printf

### luaO_chunkid
- Signature: `void luaO_chunkid(char *out, const char *source, size_t len)`
- Purpose: Create a human-readable chunk identifier from source
- Inputs: Output buffer, source name, length
- Outputs/Return: Fills output buffer
- Side effects: Writes to output buffer
- Calls: None (defined elsewhere)
- Notes: Used for debug/error messages

## Control Flow Notes
This header defines the **data model** used by all other Lua components. Type checking and value access macros are called frequently during bytecode execution (VM dispatch), garbage collection (GC marking), and API calls. The macro-heavy design allows inline checking and access without function call overhead. Initialization uses the `CommonHeader` macro pattern to ensure all GC objects are linked and tagged consistently.

## External Dependencies
- `<stdarg.h>` ΓÇô variadic argument handling
- `llimits.h` ΓÇô type limits, alignment helpers, casting macros
- `lua.h` ΓÇô public API type definitions, Lua state, type constants (LUA_TNIL, LUA_TNUMBER, etc.)
- `luaconf.h` (included indirectly via lua.h) ΓÇô configuration and platform-specific types

**Defined elsewhere:**
- `luaO_log2`, `luaO_int2fb`, `luaO_fb2int`, `luaO_rawequalObj`, `luaO_str2d`, `luaO_pushvfstring`, `luaO_pushfstring`, `luaO_chunkid` ΓÇô implemented in ldo.c or lopcodes.c
- `struct Table`, `struct Proto`, `lua_State` ΓÇô used but struct members defined here; lua_State defined in lstate.h
