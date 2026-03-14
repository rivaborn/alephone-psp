# Source_Files/Lua/lfunc.h

## File Purpose
Header declaring the Lua C API for function prototype and closure manipulation. Provides functions to create, inspect, and manage Lua function objects (both C closures wrapping native functions and Lua closures wrapping script functions).

## Core Responsibilities
- Create and allocate function prototypes (`Proto`)
- Allocate C closures (native function wrappers) and Lua closures (script function instances)
- Manage upvalues: create, find, and close them across function scopes
- Deallocate function objects and their upvalues during garbage collection
- Query function metadata (e.g., local variable names at debug locations)
- Provide sizing macros for closure memory allocation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Proto` | struct | Function prototype (bytecode, constants, nested functions, debug info) |
| `Closure` | union | Either a C closure or Lua closure; wraps a function's code and captured state |
| `CClosure` | struct | Closure holding a native C function pointer and upvalue array |
| `LClosure` | struct | Closure holding a prototype pointer and upvalue references |
| `UpVal` | struct | Reference to a variable captured by a closure (open or closed) |
| `TValue` | struct | Tagged value (type + value pair) |
| `Table` | struct | Environment table associated with a closure |

## Global / File-Static State
None.

## Key Functions / Methods

### luaF_newproto
- **Signature:** `Proto *luaF_newproto (lua_State *L)`
- **Purpose:** Allocate and initialize a new function prototype object.
- **Inputs:** `L` (Lua state)
- **Outputs/Return:** Pointer to newly allocated `Proto`
- **Side effects:** Heap allocation; object becomes GC-managed
- **Calls (direct):** Not visible in header
- **Notes:** Prototype is created before it is populated with code, constants, or nested functions.

### luaF_newCclosure
- **Signature:** `Closure *luaF_newCclosure (lua_State *L, int nelems, Table *e)`
- **Purpose:** Allocate a C closure wrapping a native function with upvalues.
- **Inputs:** `L` (Lua state), `nelems` (number of upvalues), `e` (environment table)
- **Outputs/Return:** Pointer to newly allocated `Closure`
- **Side effects:** Heap allocation; closure becomes GC-managed
- **Calls (direct):** Not visible in header
- **Notes:** `nelems` specifies the upvalue array size; the actual C function pointer is set after allocation.

### luaF_newLclosure
- **Signature:** `Closure *luaF_newLclosure (lua_State *L, int nelems, Table *e)`
- **Purpose:** Allocate a Lua closure (script function instance) with upvalue slots.
- **Inputs:** `L` (Lua state), `nelems` (number of upvalues), `e` (environment table)
- **Outputs/Return:** Pointer to newly allocated `Closure`
- **Side effects:** Heap allocation; closure becomes GC-managed
- **Calls (direct):** Not visible in header
- **Notes:** Upvalue references are populated before the closure becomes callable.

### luaF_newupval
- **Signature:** `UpVal *luaF_newupval (lua_State *L)`
- **Purpose:** Create a new upvalue object.
- **Inputs:** `L` (Lua state)
- **Outputs/Return:** Pointer to newly allocated `UpVal`
- **Side effects:** Heap allocation; upvalue becomes GC-managed
- **Calls (direct):** Not visible in header

### luaF_findupval
- **Signature:** `UpVal *luaF_findupval (lua_State *L, StkId level)`
- **Purpose:** Find or create an upvalue for a variable at a stack position.
- **Inputs:** `L` (Lua state), `level` (stack index)
- **Outputs/Return:** Pointer to upvalue (existing or newly created)
- **Side effects:** May allocate new upvalue; maintains linked list of open upvalues
- **Calls (direct):** Not visible in header
- **Notes:** If upvalue already exists at that level, returns it; otherwise creates one.

### luaF_close
- **Signature:** `void luaF_close (lua_State *L, StkId level)`
- **Purpose:** Close all open upvalues at or above a given stack level (migrate from stack to heap).
- **Inputs:** `L` (Lua state), `level` (stack index threshold)
- **Outputs/Return:** None
- **Side effects:** Modifies upvalues and the open upvalue linked list
- **Calls (direct):** Not visible in header
- **Notes:** Called when leaving a scope to finalize captured variables; typically on function return.

### luaF_freeproto
- **Signature:** `void luaF_freeproto (lua_State *L, Proto *f)`
- **Purpose:** Deallocate a function prototype and its contents.
- **Inputs:** `L` (Lua state), `f` (prototype to free)
- **Outputs/Return:** None
- **Side effects:** Heap deallocation; invalidates all closures derived from this prototype
- **Calls (direct):** Not visible in header

### luaF_freeclosure
- **Signature:** `void luaF_freeclosure (lua_State *L, Closure *c)`
- **Purpose:** Deallocate a closure (C or Lua).
- **Inputs:** `L` (Lua state), `c` (closure to free)
- **Outputs/Return:** None
- **Side effects:** Heap deallocation
- **Calls (direct):** Not visible in header

### luaF_freeupval
- **Signature:** `void luaF_freeupval (lua_State *L, UpVal *uv)`
- **Purpose:** Deallocate an upvalue object.
- **Inputs:** `L` (Lua state), `uv` (upvalue to free)
- **Outputs/Return:** None
- **Side effects:** Heap deallocation; removes from open upvalue chain if applicable
- **Calls (direct):** Not visible in header

### luaF_getlocalname
- **Signature:** `const char *luaF_getlocalname (const Proto *func, int local_number, int pc)`
- **Purpose:** Retrieve the name of a local variable in a function at a given instruction position.
- **Inputs:** `func` (prototype), `local_number` (variable index), `pc` (program counter / instruction offset)
- **Outputs/Return:** Pointer to variable name string (or `NULL` if not found)
- **Side effects:** None (read-only)
- **Calls (direct):** Not visible in header
- **Notes:** Used by debugger to map bytecode positions to variable names.

## Closure Sizing Macros

### sizeCclosure(n)
Computes heap allocation size for a `CClosure` with `n` upvalues (accounting for the base struct plus an array of `TValue`s).

### sizeLclosure(n)
Computes heap allocation size for an `LClosure` with `n` upvalues (accounting for the base struct plus an array of `UpVal*` pointers).

## Control Flow Notes
This header defines the function creation and upvalue management API used during:
1. **Compilation/Definition:** Creating `Proto` objects for each function defined in a script.
2. **Function calls:** Instantiating `Closure` objects and managing upvalue capture.
3. **Return/scope exit:** Closing open upvalues via `luaF_close`.
4. **Garbage collection:** Freeing prototypes, closures, and upvalues.
5. **Debugging:** Querying local variable names via `luaF_getlocalname`.

## External Dependencies
- **Includes:** `"lobject.h"` ΓÇö defines `Proto`, `Closure`, `UpVal`, `TValue`, `Table`, `StkId`, and all base object structures.
- **Macros used:** `LUAI_FUNC` (marks functions exported by the Lua internals), `cast()` (casts used in size macros).
- **Defined elsewhere:** `lua_State` (Lua VM state), `StkId` (stack index typedef from `lobject.h`).
