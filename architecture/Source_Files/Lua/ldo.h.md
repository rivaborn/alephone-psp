# Source_Files/Lua/ldo.h

## File Purpose
Defines the stack and call management interface for Lua's execution engine. Provides macros and function declarations for manipulating the call stack, managing function calls, and handling protected execution with error recovery.

## Core Responsibilities
- **Stack allocation & growth**: Macros for checking stack space and growing when needed
- **Stack position tracking**: Convert between stack pointers and offsets for save/restore operations
- **Call protocol**: Initiate and complete function calls (Lua and C functions)
- **Protected execution**: Wrap function calls with error handling (pcall, parser protection)
- **Error handling**: Throw exceptions and set error objects on the stack
- **Call info management**: Manage the CallInfo array (call frames)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Pfunc` | typedef (function pointer) | Protected function signature: `void (*)(lua_State *L, void *ud)` |
| `PCRLUA`, `PCRC`, `PCRYIELD` | enum constants | Return codes from `luaD_precall`: Lua call, C call, C yield |

## Global / File-Static State
None.

## Key Functions / Methods

### luaD_checkstack (macro)
- **Signature**: `luaD_checkstack(L, n)`
- **Purpose**: Ensure `n` stack slots are available; grow if necessary
- **Inputs**: `L` (lua_State), `n` (slots needed)
- **Side effects**: May reallocate stack; calls `luaD_growstack()` or `luaD_reallocstack()`
- **Notes**: Inline macro for hot-path performance

### incr_top (macro)
- **Signature**: `incr_top(L)`
- **Purpose**: Safely increment stack top by one slot
- **Side effects**: Checks stack before incrementing
- **Calls**: `luaD_checkstack()`

### savestack / restorestack (macros)
- **Purpose**: Convert between stack pointer and offset for safe stack reallocation
- **Inputs**: Stack pointer or offset
- **Outputs**: Offset or pointer
- **Notes**: Used when stack may be reallocated during execution

### luaD_precall
- **Signature**: `int luaD_precall(lua_State *L, StkId func, int nresults)`
- **Purpose**: Prepare a function call; determine if Lua or C function
- **Outputs**: Returns `PCRLUA`, `PCRC`, or `PCRYIELD`
- **Side effects**: Sets up call frame (CallInfo)

### luaD_call
- **Signature**: `void luaD_call(lua_State *L, StkId func, int nResults)`
- **Purpose**: Execute a function call (unprotected)
- **Inputs**: Function object and expected result count

### luaD_pcall
- **Signature**: `int luaD_pcall(lua_State *L, Pfunc func, void *u, ptrdiff_t oldtop, ptrdiff_t ef)`
- **Purpose**: Protected call with error recovery
- **Inputs**: Function to run, user data, saved stack position, error function index
- **Outputs**: Error code (0 = success, nonzero = error)

### luaD_throw / luaD_rawrunprotected
- **Purpose**: Throw exceptions and low-level protected execution
- **Notes**: Implements setjmp/longjmp-style error handling

## Control Flow Notes
**Stack management**: Called during init to set up stack; continuously used during frame/update to manage function calls.  
**Call execution**: `precall` ΓåÆ decide Lua/C ΓåÆ execute ΓåÆ `poscall` (cleanup and place results).  
**Error handling**: Protected variants (`pcall`, `protectedparser`) catch errors via `luaD_throw` for graceful recovery.

## External Dependencies
- **lobject.h**: `TValue`, `StkId` (stack value type and stack index)
- **lstate.h**: `lua_State`, `CallInfo`, `global_State` (execution state)
- **lzio.h**: `ZIO` (buffered input for parser)
