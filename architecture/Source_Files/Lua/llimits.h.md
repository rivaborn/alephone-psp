# Source_Files/Lua/llimits.h

## File Purpose

Core infrastructure header for Lua that defines installation-dependent limits, type aliases, and utility macros. Establishes portable integer types, memory bounds, VM instruction encoding, and assertion/debugging infrastructure used throughout the Lua engine.

## Core Responsibilities

- Define portable integer types (lu_int32, lu_mem, l_mem, lu_byte) abstracted via configuration
- Establish hard limits for stack depth, string tables, and memory allocation
- Provide type-casting macros and assertion helpers for safety and debugging
- Define the Instruction type (VM instruction encoding, 32-bit unsigned)
- Configure thread-safety primitives (lua_lock, lua_unlock, threadyield)
- Provide pointer-to-integer hashing conversion (IntPoint)
- Define stack and buffer size thresholds (MAXSTACK, MINSTRTABSIZE, LUA_MINBUFFER)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| lu_int32 | typedef | 32-bit unsigned integer for portable cross-platform use |
| lu_mem | typedef | Unsigned memory size type (abstracted from LUAI_UMEM) |
| l_mem | typedef | Signed memory type (abstracted from LUAI_MEM) |
| lu_byte | typedef | unsigned char; used for small integers, reserves char for characters |
| L_Umaxalign | typedef | Ensures maximum alignment (platform-dependent via LUAI_USER_ALIGNMENT_T) |
| l_uacNumber | typedef | Result type of "usual argument conversion" for lua_Number values |
| Instruction | typedef | VM instruction type; must be unsigned 32-bit (see lopcodes.h) |

## Global / File-Static State

None.

## Key Functions / Methods

None (header file; only macros and typedefs).

### Macro Summary
- **Casting helpers**: `cast()`, `cast_byte()`, `cast_num()`, `cast_int()` ΓÇö unsafe type coercion with explicit notation
- **Assertion control**: `lua_assert()`, `check_exp()`, `api_check()` ΓÇö conditional and no-op variants depending on build
- **Utilities**: `UNUSED()` ΓÇö suppresses compiler warnings for intentional unused parameters; `IntPoint(p)` ΓÇö pointer-to-int hash
- **Thread safety**: `lua_lock()`, `lua_unlock()`, `luai_threadyield()` ΓÇö default to no-ops; overridable for MT builds
- **Stack validation**: `condhardstacktests()` ΓÇö conditional debug testing on stack reallocation

## Control Flow Notes

Purely a configuration header; no control flow. Establishes constants and type definitions loaded by all Lua engine modules. Used at compile time for conditional compilation and at runtime for bound-checking (e.g., MAX_SIZET, MAX_INT guard against overflow).

## External Dependencies

- **Standard library**: `<limits.h>` (INT_MAX), `<stddef.h>` (size_t, NULL)
- **Lua headers**: `"lua.h"` (version, type constants, lua_State, function pointers)
- **Config abstraction**: References macros from `luaconf.h` (included transitively via lua.h): LUAI_UINT32, LUAI_UMEM, LUAI_MEM, LUAI_USER_ALIGNMENT_T, LUAI_UACNUMBER
- **Defined elsewhere**: lua_State (opaque in this file, declared in lua.h)
