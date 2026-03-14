# Source_Files/Lua/lapi.h

## File Purpose
Header file declaring auxiliary API functions for Lua's internal object manipulation. Provides a single entry point to push Lua values onto the interpreter stack, bridging internal object representation with stack-based API operations.

## Core Responsibilities
- Declare auxiliary API functions for object stack operations
- Provide minimal interface between Lua's internal TValue representation and public stack APIs

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaA_pushobject
- **Signature:** `void luaA_pushobject (lua_State *L, const TValue *o)`
- **Purpose:** Push a Lua object onto the stack of a given state
- **Inputs:**
  - `lua_State *L`: Target Lua interpreter state
  - `const TValue *o`: Pointer to the tagged value to push (contains type tag + union value)
- **Outputs/Return:** void (modifies stack in-place)
- **Side effects:** Advances stack pointer in the Lua state; may trigger GC or memory operations
- **Calls:** Not inferable from this file (defined elsewhere)
- **Notes:** `LUAI_FUNC` marker indicates internal API (not part of public Lua C API). Typically called when C code needs to move values between internal representations and the Lua stack.

## Control Flow Notes
This is an auxiliary utility called during stack manipulation within the interpreter's internal C API. Likely invoked by other internal functions when pushing values during execution, object copying, or stack restructuring operations.

## External Dependencies
- **Includes:** `"lobject.h"` ΓÇö provides `TValue`, `lua_State` definitions and type system
- **Macros used:** `LUAI_FUNC` (from `llimits.h`, included transitively) ΓÇö marks internal function declarations
