# Source_Files/Lua/lualib.h

## File Purpose
Header file declaring the Lua standard library initialization API. Defines constants for library names, declares functions to open individual standard libraries (base, table, io, os, string, math, debug, package), and provides a convenience function to open all libraries at once.

## Core Responsibilities
- Declare library opener functions (`luaopen_*`) for each standard library
- Define symbolic constants for library names (`LUA_*LIBNAME`)
- Define the file handle type key constant (`LUA_FILEHANDLE`)
- Declare the bulk library opener (`luaL_openlibs`)
- Provide assertion macro (`lua_assert`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaopen_base
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize and register the base library (core language functions like `print`, `type`, `error`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code (0 for success; non-zero on error)
- Side effects: Pushes library table onto stack, registers globals
- Calls: Implementation in `lualib.c` (not visible here)
- Notes: Symbolic name is `LUA_COLIBNAME` (`"coroutine"`); likely a misnamingΓÇöthis opens the base library, not coroutines

### luaopen_table
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize and register the table manipulation library (`table.insert`, `table.remove`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers table library functions
- Calls: Implementation in `lualib.c`

### luaopen_io
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize file I/O library (`io.open`, `io.read`, `io.write`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers I/O functions; may establish stdin/stdout/stderr
- Calls: Implementation in `lualib.c`

### luaopen_os
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize OS interface library (`os.time`, `os.getenv`, `os.execute`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers OS interaction functions
- Calls: Implementation in `lualib.c`

### luaopen_string
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize string manipulation library (`string.sub`, `string.format`, `string.match`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers string functions
- Calls: Implementation in `lualib.c`

### luaopen_math
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize math library (`math.sin`, `math.sqrt`, `math.random`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers math functions
- Calls: Implementation in `lualib.c`

### luaopen_debug
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize debugging library (`debug.getinfo`, `debug.setlocal`, `debug.sethook`, etc.)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers debugging functions
- Calls: Implementation in `lualib.c`

### luaopen_package
- Signature: `int (*)(lua_State *L)`
- Purpose: Initialize package/module loading library (`require`, module path configuration)
- Inputs: Lua state
- Outputs/Return: Status code
- Side effects: Registers package system; sets up module search paths
- Calls: Implementation in `lualib.c`

### luaL_openlibs
- Signature: `void (*)(lua_State *L)`
- Purpose: Convenience function to open all standard libraries in one call
- Inputs: Lua state
- Outputs/Return: None
- Side effects: Calls all `luaopen_*` functions internally
- Calls: `luaopen_base`, `luaopen_table`, `luaopen_io`, `luaopen_os`, `luaopen_string`, `luaopen_math`, `luaopen_debug`, `luaopen_package` (implementation detail)
- Notes: Typical initialization path for embedding Lua

## Control Flow Notes
This is an include-only declarations file with no control flow of its own. Integration pattern:
1. Caller includes this header
2. Caller creates a `lua_State` (via `lua_newstate`)
3. Caller invokes `luaL_openlibs()` or specific `luaopen_*` functions to initialize libraries
4. Lua code then has access to registered library functions

## External Dependencies
- **Includes:** `lua.h` (core Lua API, defines `lua_State`, constants, stack manipulation)
- **Macros used:** `LUALIB_API` (visibility/calling convention, defined in `luaconf.h`)
- **Types used (from lua.h):** `lua_State`
