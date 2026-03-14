# Source_Files/Lua/lvm.h

## File Purpose
Header file for the Lua virtual machine (VM). Declares the core VM operations including bytecode execution, type comparisons, type conversions, and table element access. Serves as the public interface to VM primitives used throughout the interpreter.

## Core Responsibilities
- Bytecode execution via the main VM dispatch loop
- Type comparison operations (less-than, equality)
- Type coercion and conversion (to number, to string)
- Table element access (get/set operations)
- String concatenation with type coercion
- Convenience macros for runtime type checking and conversion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TValue` | struct (from lobject.h) | Tagged value: a union holding any Lua value plus its type tag |
| `StkId` | typedef | Stack identifier; alias for `TValue*` pointing to stack slots |
| `lua_State` | struct (opaque here) | Lua interpreter state; required for all VM operations |

## Global / File-Static State
None.

## Key Functions / Methods

### luaV_execute
- **Signature:** `void luaV_execute (lua_State *L, int nexeccalls);`
- **Purpose:** Main bytecode interpreter loop; executes Lua bytecode instructions.
- **Inputs:** Lua state `L`, recursion depth counter `nexeccalls`.
- **Outputs/Return:** None (modifies state via side effects).
- **Side effects:** Modifies stack, call frames, and global state; may trigger GC.
- **Calls:** Not visible in this header; implemented in lvm.c.
- **Notes:** Core VM dispatch routine; critical path for all Lua code execution.

### luaV_lessthan
- **Signature:** `int luaV_lessthan (lua_State *L, const TValue *l, const TValue *r);`
- **Purpose:** Implements the `<` comparison operator with type coercion and metamethods.
- **Inputs:** Two tagged values to compare.
- **Outputs/Return:** 1 if `l < r`, 0 otherwise.
- **Side effects:** May invoke tag methods (metamethods); may raise errors.
- **Calls:** Uses tag method system (defined elsewhere).
- **Notes:** Handles mixed-type comparisons and `__lt` metamethod dispatch.

### luaV_equalval
- **Signature:** `int luaV_equalval (lua_State *L, const TValue *t1, const TValue *t2);`
- **Purpose:** Implements the `==` operator with tag method support.
- **Inputs:** Two tagged values to compare.
- **Outputs/Return:** 1 if equal, 0 otherwise.
- **Side effects:** May invoke `__eq` metamethods; may raise errors.
- **Calls:** Tag method lookup (via ltm.h).
- **Notes:** Raw equality differs from metamethod equality.

### luaV_tonumber
- **Signature:** `const TValue *luaV_tonumber (const TValue *obj, TValue *n);`
- **Purpose:** Attempts to convert a value to a number; returns pointer to numeric result.
- **Inputs:** Source value `obj`, output buffer `n`.
- **Outputs/Return:** Pointer to numeric value (either original or `n`), or NULL if conversion fails.
- **Side effects:** None (pure conversion, no state modification).
- **Calls:** None visible.
- **Notes:** Used by arithmetic operators and type coercion; no metamethod support.

### luaV_tostring
- **Signature:** `int luaV_tostring (lua_State *L, StkId obj);`
- **Purpose:** Attempts to convert a stack value to a string in-place; invokes `__tostring` metamethod if applicable.
- **Inputs:** Lua state `L`, stack position `obj`.
- **Outputs/Return:** 1 on success (value is now a string), 0 on failure.
- **Side effects:** Modifies the value at `obj` on the stack; may allocate strings; may invoke metamethods.
- **Calls:** Tag method system; string allocation (defined elsewhere).
- **Notes:** Modifies the stack in-place; used by `print()`, concatenation, and debug operations.

### luaV_gettable
- **Signature:** `void luaV_gettable (lua_State *L, const TValue *t, TValue *key, StkId val);`
- **Purpose:** Retrieves a table element with metamethod support (for `t[key]`).
- **Inputs:** Table value `t`, key, output location `val`.
- **Outputs/Return:** Result stored at `val`.
- **Side effects:** May invoke `__index` metamethods; stack manipulation.
- **Calls:** Table lookup, tag methods (defined elsewhere).
- **Notes:** Handles both array and hash parts; supports `__index` chain.

### luaV_settable
- **Signature:** `void luaV_settable (lua_State *L, const TValue *t, TValue *key, StkId val);`
- **Purpose:** Sets a table element with metamethod support (for `t[key] = val`).
- **Inputs:** Table value `t`, key, value `val` at stack position.
- **Outputs/Return:** None.
- **Side effects:** Modifies table; may invoke `__newindex` metamethods; may trigger GC.
- **Calls:** Table update, tag methods (defined elsewhere).
- **Notes:** Handles `__newindex` chains; may fail with errors from metamethods.

### luaV_concat
- **Signature:** `void luaV_concat (lua_State *L, int total, int last);`
- **Purpose:** Concatenates `total` stack values starting at index `last`, with type coercion.
- **Inputs:** Count of values to concatenate, stack index of first value.
- **Outputs/Return:** Result replaces stack values; new string on stack.
- **Side effects:** Allocates string, modifies stack, may invoke `__concat` metamethods.
- **Calls:** String allocation, type coercion, tag methods.
- **Notes:** Right-associative; coerces numbers to strings.

## Macros

### Convenience Macros
- **`tostring(L, o)`** ΓåÆ `(ttype(o) == LUA_TSTRING) || luaV_tostring(L, o)` ΓÇö Check if already string or convert.
- **`tonumber(o, n)`** ΓåÆ Converts `o` to number via `luaV_tonumber()`, stores in `n`; returns success.
- **`equalobj(L, o1, o2)`** ΓåÆ Alias for `luaV_equalval()` with metamethods.

## Control Flow Notes
**Execution model:** `luaV_execute()` is the main entry point for bytecode interpretation. It dispatches individual VM instructions. Type operations (`tonumber`, `tostring`, comparisons) are invoked by VM instructions during execution. Table operations (`gettable`, `settable`) handle Lua's table indexing semantics. Metamethod support threads through all value operations via the tag method system (ltm.h).

## External Dependencies
- **`ldo.h`** ΓÇö Stack and call frame management (stack growth, call protocol).
- **`lobject.h`** ΓÇö Type definitions for Lua values, tagged unions, string/table/function structures.
- **`ltm.h`** ΓÇö Tag method (metamethod) dispatch and lookup.
