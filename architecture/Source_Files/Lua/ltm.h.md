# Source_Files/Lua/ltm.h

## File Purpose
Defines Lua's tag method (metamethod) system, which implements operator overloading and special behaviors for Lua objects. Declares the enumeration of all tag method types and provides macros and functions for efficient tag method lookup during runtime.

## Core Responsibilities
- Define the `TMS` enum cataloging all supported tag methods (INDEX, NEWINDEX, GC, arithmetic ops, etc.)
- Provide optimized macros (`gfasttm`, `fasttm`) for fast tag method lookups with caching via flags
- Declare functions for tag method retrieval from tables and objects
- Initialize the tag method subsystem at Lua startup
- Separate fast-path methods (INDEX, NEWINDEX, GC, MODE, EQ) from regular methods for performance

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TMS` | enum | Enumerates all 17 tag method types; `TM_N` marks the count |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luaT_typenames` | `const char *const[]` | global | Maps type tags to their string names |

## Key Functions / Methods

### luaT_gettm
- Signature: `const TValue *luaT_gettm(Table *events, TMS event, TString *ename)`
- Purpose: Retrieve a tag method from an event table by event type
- Inputs: `events` (metatable), `event` (tag method type), `ename` (method name string)
- Outputs/Return: Pointer to the tag method value, or NULL if not found
- Calls: (defined elsewhere)

### luaT_gettmbyobj
- Signature: `const TValue *luaT_gettmbyobj(lua_State *L, const TValue *o, TMS event)`
- Purpose: Look up a tag method from an object's metatable
- Inputs: `L` (Lua state), `o` (object value), `event` (tag method type)
- Outputs/Return: Pointer to the tag method, or NULL if not found
- Calls: (defined elsewhere)

### luaT_init
- Signature: `void luaT_init(lua_State *L)`
- Purpose: Initialize the tag method subsystem at Lua startup
- Inputs: `L` (Lua state)
- Outputs/Return: None
- Side effects: Populates global tag method name table
- Calls: (defined elsewhere)

## Control Flow Notes
Tag methods are queried during operator evaluation, table access (INDEX/NEWINDEX), garbage collection (GC), and function calls (CALL). The `fasttm` macro optimizes repeated lookups by checking the table's `flags` bitfield to cache "method not present" state, avoiding redundant table lookups in hot paths.

## External Dependencies
- `lobject.h`: defines `TValue`, `Table`, `TString`, and related macros
- Lua core state structures (`lua_State`, accessed via `G(l)` macro)
- Tag method declarations implemented elsewhere (likely `ltm.c`)
