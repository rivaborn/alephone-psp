# Source_Files/Lua/lauxlib.h

## File Purpose
Header file declaring the Lua Auxiliary Library API for C code building Lua libraries and extensions. Provides registration, type checking, error handling, and utility functions for integrating C code with Lua 5.1.

## Core Responsibilities
- Declare library registration and initialization functions
- Provide type checking and argument validation helpers
- Declare error reporting and stack manipulation functions
- Declare string/number/integer conversion and validation functions
- Provide buffer management for efficient string concatenation
- Declare reference management (garbage collection protection)
- Declare code loading functions (from files, strings, buffers)
- Provide compatibility macros for different Lua versions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `luaL_Reg` | struct | Maps C function names to `lua_CFunction` pointers for library registration |
| `luaL_Buffer` | struct | Manages dynamic string buffer with position tracking, stack level, and fixed backing store |

## Global / File-Static State
None.

## Key Functions / Methods

### luaL_register / luaI_openlib
- Signature: `void luaL_register(lua_State *L, const char *libname, const luaL_Reg *l)` / `void luaI_openlib(lua_State *L, const char *libname, const luaL_Reg *l, int nup)`
- Purpose: Register a C library with Lua by mapping function names to C closures
- Inputs: Lua state, library name (optional), array of name/function pairs, optional upvalue count
- Outputs/Return: None
- Side effects: Modifies Lua stack and globals; creates/updates library table
- Notes: `luaL_register` wraps `luaI_openlib` with 0 upvalues

### luaL_newstate
- Signature: `lua_State *(luaL_newstate)(void)`
- Purpose: Allocate and initialize a new Lua interpreter state
- Inputs: None
- Outputs/Return: Pointer to new `lua_State`
- Side effects: Allocates memory; initializes VM
- Calls: Allocates via default allocator

### Type Checking Functions (luaL_checkstring, luaL_checkinteger, luaL_checknumber, etc.)
- Purpose: Validate stack argument type and raise error if invalid
- Inputs: Lua state, argument index, (optional) error message or typename
- Outputs/Return: Converted value (string, number, integer) or userdata pointer
- Side effects: Raises `luaL_error` on type mismatch
- Notes: `luaL_optnumber`, `luaL_optinteger`, etc. accept default values for nil/missing args; macros like `luaL_checkstring` wrap `luaL_checklstring` with `NULL` length pointer

### Error & Validation (luaL_error, luaL_argerror, luaL_typerror)
- Purpose: Raise Lua errors with formatted messages
- Inputs: Lua state, argument index, message/format string
- Outputs/Return: Never returns (jumps via `longjmp`)
- Side effects: Throws exception-like error to caller
- Notes: `luaL_argerror` reports bad argument; `luaL_where` adds call-site info

### Reference Management (luaL_ref, luaL_unref)
- Purpose: Create/destroy protected references to Lua objects (prevent GC)
- Inputs: Lua state, table index (e.g., `LUA_REGISTRYINDEX`), reference ID
- Outputs/Return: Unique reference integer or nil
- Side effects: Stores/retrieves object in registry
- Notes: `lua_ref` / `lua_unref` are compatibility wrappers

### Code Loading (luaL_loadfile, luaL_loadstring, luaL_loadbuffer)
- Purpose: Load Lua code from file, string, or buffer into stack
- Inputs: Lua state, filename/string/buffer, optional size and name
- Outputs/Return: 0 on success; error code (`LUA_ERRFILE`, etc.) on failure
- Side effects: Pushes compiled function onto stack or error message
- Notes: Does not execute; caller must `lua_pcall` to run

### Buffer Functions (luaL_buffinit, luaL_addlstring, luaL_pushresult)
- Purpose: Accumulate strings efficiently without repeated allocations
- Inputs: Lua state, buffer, string data and length
- Outputs/Return: `luaL_prepbuffer` returns pointer to writable area; `luaL_pushresult` pushes final string
- Side effects: Modifies internal buffer; may trigger flush to stack
- Calls: `luaL_prepbuffer` expands buffer when full

### Meta & Option Handling (luaL_getmetafield, luaL_callmeta, luaL_checkoption)
- Purpose: Invoke metamethods and validate option strings
- Inputs: Lua state, object index, field/method name, option list
- Outputs/Return: Metatable field or result of metamethod call; option index
- Side effects: Stack manipulation
- Notes: `luaL_callmeta` safely calls metamethods with error handling

## Control Flow Notes
This is a header-only interface; control flow is determined by caller. Typical usage:
1. `luaL_newstate()` initializes VM
2. `luaL_register()` sets up C library functions
3. C functions call `luaL_check*` to validate arguments and `luaL_error()` to report errors
4. `luaL_ref()` / `luaL_unref()` manage object lifetime across calls
5. Buffer functions accumulate results before returning to Lua

## External Dependencies
- Includes: `<stddef.h>`, `<stdio.h>`, `lua.h`
- Depends on: `lua_State`, `lua_CFunction`, `lua_Number`, `lua_Integer`, `LUALIB_API`, `LUA_REGISTRYINDEX`, `lua_pcall`, `lua_error` (from `lua.h`)
- Defines compatibility wrappers for `LUA_COMPAT_GETN`, `LUA_COMPAT_OPENLIB` based on configuration
