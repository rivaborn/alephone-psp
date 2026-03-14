# Source_Files/Lua/lstate.h

## File Purpose
Core header defining Lua's execution state structures. Declares the global state shared across all threads, per-thread execution state, call stack frames, and provides type-conversion macros for garbage-collectable objects. Manages thread creation and destruction.

## Core Responsibilities
- Define `global_State` structure for VM-wide shared state (memory allocation, GC, string interning, registry)
- Define `lua_State` structure for per-thread execution context (stack, call info, hooks, upvalues)
- Define `CallInfo` for tracking function call frames on the call stack
- Define `stringtable` hash table for string interning and deduplication
- Provide accessor macros (`G()`, `gt()`, `registry()`, `curr_func()`) for navigating state
- Define `GCObject` union encompassing all garbage-collectable types
- Provide type-casting macros (`gco2ts()`, `gco2h()`, `gco2cl()`, etc.) for safe GCObject conversion

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `stringtable` | struct | Hash table mapping strings to interned TString objects; tracks `nuse` (element count) and size |
| `CallInfo` | struct | Call frame metadata: function base/top, saved PC, expected results, tail-call tracking |
| `global_State` | struct | Shared VM state: GC state/lists, string table, allocator callback, registry table, metatables, tag-method names |
| `lua_State` | struct | Per-thread state: execution stack, call-info array, status, hooks, globals table, error recovery point |
| `GCObject` | union | Supertype for all collectable objects: GCheader, TString, Udata, Closure, Table, Proto, UpVal, lua_State |

## Global / File-Static State
None.

## Key Functions / Methods

### luaE_newthread
- Signature: `lua_State *luaE_newthread (lua_State *L)`
- Purpose: Create a new coroutine/thread within the VM
- Inputs: Pointer to existing `lua_State` (to access global state and allocator)
- Outputs/Return: Pointer to newly allocated `lua_State`
- Side effects: Allocates memory via `global_State.frealloc`; new thread shares `global_State` with parent
- Calls: Not visible in this header (defined in ldo.c)
- Notes: New thread has its own stack, call-info array, and hook state but shares the global state (GC, registry, strings)

### luaE_freethread
- Signature: `void luaE_freethread (lua_State *L, lua_State *L1)`
- Purpose: Deallocate a thread's per-thread structures
- Inputs: `L` (current thread context), `L1` (thread to free)
- Outputs/Return: None
- Side effects: Deallocates stack and CallInfo array via global allocator
- Calls: Not visible in this header (defined in ldo.c)
- Notes: Does not free the global state; global state is freed when the main thread is closed

## Control Flow Notes
This file defines static data layout and does not directly participate in execution flow. However:
- **Initialization**: Threads are created by `luaE_newthread`; global state is allocated during `lua_newstate` (lua.h).
- **Per-frame**: Each function call pushes a `CallInfo` entry; `curr_func(L)` retrieves the current function via `L->ci`.
- **GC**: `global_State` maintains GC lists (rootgc, gray, weak) traversed during mark-and-sweep.
- **Shutdown**: Threads freed via `luaE_freethread`; global state freed via `lua_close` (lua.h).

## External Dependencies
- **Includes:** `lua.h` (Lua version and type definitions), `lobject.h` (TValue, GCheader, Closure, Proto, Table, UpVal, TString, Udata), `ltm.h` (tag-method enumeration), `lzio.h` (Mbuffer)
- **Defined elsewhere:** `struct lua_longjmp` (ldo.c), conversion functions and GC internals
