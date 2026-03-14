# Source_Files/Lua/lmem.h

## File Purpose
Memory manager interface header for Lua 5.1. Provides a macro-based abstraction over allocation, deallocation, and reallocation operations with built-in overflow checking and a unified error path through `lua_State`.

## Core Responsibilities
- Define allocation macros for single objects (`luaM_new`), arrays (`luaM_newvector`), and dynamic growth (`luaM_growvector`)
- Define deallocation macros for flexible cleanup patterns (`luaM_free`, `luaM_freemem`, `luaM_freearray`)
- Protect against integer overflow when computing allocation sizes via `luaM_reallocv` size checks
- Provide a standard error message constant for memory exhaustion
- Route all memory operations through a central `lua_State` context for error propagation

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaM_realloc_
- **Signature:** `void *luaM_realloc_ (lua_State *L, void *block, size_t oldsize, size_t size)`
- **Purpose:** Core memory reallocation handler for all allocation, resize, and deallocation requests
- **Inputs:** Lua state, existing block pointer, old size, new size (0 = deallocate)
- **Outputs/Return:** New block pointer, or error propagated via `lua_State`
- **Side effects:** Heap modification; may trigger GC via state
- **Calls:** Invoked indirectly through all user-facing allocation macros
- **Notes:** Implementation in `lmem.c`. All paths converge here.

### luaM_toobig
- **Signature:** `void *luaM_toobig (lua_State *L)`
- **Purpose:** Error handler when allocation size exceeds `MAX_SIZET`
- **Inputs:** Lua state (for error reporting)
- **Outputs/Return:** Does not return; raises memory error
- **Side effects:** Throws exception
- **Calls:** Invoked by `luaM_reallocv` macro when overflow check fails
- **Notes:** Safety valve to prevent wrapping on malformed size calculations

### luaM_growaux_
- **Signature:** `void *luaM_growaux_ (lua_State *L, void *block, int *size, size_t size_elem, int limit, const char *errormsg)`
- **Purpose:** Exponential-growth helper for dynamic arrays (used by `luaM_growvector`)
- **Inputs:** State, block, pointer to element count, element size, max limit, error message
- **Outputs/Return:** Reallocated block; updates `*size` to new capacity
- **Side effects:** Memory reallocation; mutates `size` argument
- **Calls:** Invoked by `luaM_growvector` macro
- **Notes:** Reduces reallocation overhead by growing more aggressively than 1x per call

## Control Flow Notes
This is a header-only interface. Macros expand at call sites throughout Lua's runtime:
- Allocation during interpreter init and object construction (tables, strings, functions)
- Growth when containers (arrays, stacks) exceed capacity
- Deallocation during GC and shutdown

The conditional logic in `luaM_reallocv` (size overflow check) gates safe allocation; oversized requests branch to `luaM_toobig` instead of passing corrupt size calculations downstream.

## External Dependencies
- **Includes:**
  - `<stddef.h>` ΓÇô `size_t` type
  - `"llimits.h"` ΓÇô `MAX_SIZET`, `cast` macro
  - `"lua.h"` ΓÇô `lua_State` struct, error constants
- **Defined elsewhere:**
  - `lua_State`: Interpreter state context
  - `MAX_SIZET`: Safe upper bound for size_t (defined in `llimits.h`)
