# Source_Files/Lua/lua.h

## File Purpose
Public C API header for Lua 5.1.2, an extensible extension language. Defines the complete interface for embedding Lua in C/C++ applications, enabling stack-based value access, function calls, coroutine management, and debugging hooks.

## Core Responsibilities
- Define opaque `lua_State` type and function pointer signatures (`lua_CFunction`, `lua_Reader`, `lua_Writer`, `lua_Alloc`, `lua_Hook`)
- Declare Lua type constants (`LUA_TNIL`, `LUA_TBOOLEAN`, `LUA_TNUMBER`, etc.) and error codes
- Provide stack manipulation functions for transferring data between Lua and C
- Enable C code to call Lua functions and vice versa
- Define table/userdata access functions for object manipulation
- Support coroutines, garbage collection control, and debugging hooks
- Supply utility macros for common operations (type checks, stack operations)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `lua_State` | opaque struct (pointer) | Represents a Lua execution state/VM instance |
| `lua_CFunction` | function pointer typedef | Signature for C functions callable from Lua: `int (*)(lua_State*)` |
| `lua_Reader` | function pointer typedef | Callback for custom chunk loading |
| `lua_Writer` | function pointer typedef | Callback for custom chunk dumping |
| `lua_Alloc` | function pointer typedef | Memory allocation function for custom allocators |
| `lua_Hook` | function pointer typedef | Debugger hook callback: `void (*)(lua_State*, lua_Debug*)` |
| `lua_Debug` | struct | Debug activation record; contains event, name, source, line number, upvalue count |
| `lua_Number` | typedef | Numeric type (configured as `double` in luaconf.h) |
| `lua_Integer` | typedef | Integer type (configured as `ptrdiff_t` in luaconf.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### lua_newstate
- Signature: `lua_State *(lua_newstate)(lua_Alloc f, void *ud)`
- Purpose: Create a new Lua state with custom memory allocator
- Inputs: `f` (allocator function), `ud` (opaque data for allocator)
- Outputs/Return: Pointer to new `lua_State`, or NULL on failure
- Side effects: Allocates memory; initializes global state
- Calls: Custom allocator `f`
- Notes: Entry point for embedding Lua; all further operations on that state go through the returned pointer

### lua_close
- Signature: `void (lua_close)(lua_State *L)`
- Purpose: Close a Lua state and free all memory
- Inputs: `L` (state to close)
- Outputs/Return: None
- Side effects: Deallocates all Lua objects, triggers finalizers
- Calls: Registered `lua_Alloc` for deallocation

### lua_gettop / lua_settop
- Signature: `int (lua_gettop)(lua_State *L)` / `void (lua_settop)(lua_State *L, int idx)`
- Purpose: Get/set stack top (count of elements on stack)
- Inputs: `L` (state), `idx` (target stack position for settop)
- Outputs/Return: Current top for gettop; none for settop
- Side effects: settop pops/discards elements or pushes nils to reach target depth
- Notes: Core for stack frame management; `settop(-1)` pops one element

### lua_pushvalue
- Signature: `void (lua_pushvalue)(lua_State *L, int idx)`
- Purpose: Push a copy of stack element at index `idx`
- Inputs: `L` (state), `idx` (source stack index)
- Outputs/Return: None
- Side effects: Increments stack top, duplicates reference on stack
- Notes: Index can be negative (relative to top)

### lua_call / lua_pcall
- Signature: `void (lua_call)(lua_State *L, int nargs, int nresults)` / `int (lua_pcall)(lua_State *L, int nargs, int nresults, int errfunc)`
- Purpose: Call a Lua function; `lua_pcall` includes protected error handling
- Inputs: `L` (state), `nargs` (number of arguments on stack), `nresults` (number of results to keep), `errfunc` (error handler index for pcall)
- Outputs/Return: `lua_call` returns void; `lua_pcall` returns status code (0=success, error code otherwise)
- Side effects: Pops function and arguments; pushes results
- Notes: Function must be on top of stack before call; `nresults` can be `LUA_MULTRET` (-1) to keep all results

### lua_gettable / lua_setfield
- Signature: `void (lua_gettable)(lua_State *L, int idx)` / `void (lua_setfield)(lua_State *L, int idx, const char *k)`
- Purpose: Retrieve/store value in table at index `idx` with key
- Inputs: `L` (state), `idx` (table index), `k` (string key)
- Outputs/Return: None
- Side effects: `gettable` pops key, pushes result; `setfield` pops value
- Notes: `lua_gettable` requires key on stack; `lua_setfield` uses literal string key

### lua_createtable
- Signature: `void (lua_createtable)(lua_State *L, int narr, int nrec)`
- Purpose: Create a new empty table
- Inputs: `L` (state), `narr` (array slots hint), `nrec` (record slots hint)
- Outputs/Return: None
- Side effects: Pushes new table on stack
- Notes: Hints optimize pre-allocation; not binding

### lua_next
- Signature: `int (lua_next)(lua_State *L, int idx)`
- Purpose: Iterate over table at index `idx`; yields next key-value pair
- Inputs: `L` (state), `idx` (table index)
- Outputs/Return: 0 if done, 1 if key-value pushed
- Side effects: Pops previous key; pushes next key and value (or nothing if done)
- Notes: Start iteration by pushing `nil` as first key; used in `for k,v in pairs(t)` loops

### lua_gc
- Signature: `int (lua_gc)(lua_State *L, int what, int data)`
- Purpose: Control garbage collection behavior
- Inputs: `L` (state), `what` (GC command: `LUA_GCCOLLECT`, `LUA_GCSTOP`, etc.), `data` (parameter)
- Outputs/Return: Varies by command (e.g., memory count)
- Side effects: Triggers collection, changes GC state
- Notes: Commands include stop, restart, single step, set pause/stepmul

### lua_getstack / lua_getinfo
- Signature: `int (lua_getstack)(lua_State *L, int level, lua_Debug *ar)` / `int (lua_getinfo)(lua_State *L, const char *what, lua_Debug *ar)`
- Purpose: Debug API; retrieve stack frame and function info
- Inputs: `L` (state), `level` (stack depth), `what` (info mask: 'S'=source, 'l'=line, 'n'=name, 'u'=upvalues)
- Outputs/Return: 1 if found, 0 otherwise
- Side effects: Populates `lua_Debug` struct
- Notes: Used by debuggers and profiling tools

---

## Control Flow Notes
This is a header-only interface. Typical game engine usage:
1. **Init phase**: Call `lua_newstate()` to create state, load scripts
2. **Runtime/frame**: Call Lua functions and access Lua data via stack API
3. **Shutdown**: Call `lua_close()` to clean up

The stack is the central data transfer mechanism: C pushes/pops values, calls Lua functions, inspects results.

## External Dependencies
- `<stdarg.h>` ΓÇö for `va_list` in `lua_pushvfstring()`
- `<stddef.h>` ΓÇö for `size_t`
- `"luaconf.h"` ΓÇö configuration defines (`LUA_NUMBER`, `LUA_INTEGER`, `LUA_API`, type constants, memory limits)
