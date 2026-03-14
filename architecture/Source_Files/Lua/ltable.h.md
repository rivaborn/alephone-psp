# Source_Files/Lua/ltable.h

## File Purpose
Public interface header for Lua's hash table implementation. Declares functions for table creation, value lookup/insertion, and iteration. Provides utility macros for accessing hash table node structures.

## Core Responsibilities
- Define accessor macros for hash table nodes (`gnode`, `gkey`, `gval`, `gnext`)
- Declare public table API functions (create, get, set, resize, iterate)
- Support multiple key types: numeric, string, and generic `TValue` keys
- Provide table size queries and free operations
- Expose debug utilities for internal table structure inspection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Table` | struct (from lobject.h) | Main table structure: contains array part, hash part (node array), metatable, and GC linkage |
| `Node` | struct (from lobject.h) | Hash table entry: holds value and key; used in hash chain |
| `TKey` | union (from lobject.h) | Tagged key: either full TValue+chaining info or plain TValue |
| `TValue` | struct (from lobject.h) | Tagged value: union of types (number, string, table, etc.) with type tag |
| `TString` | union (from lobject.h) | Lua string object; used as key type |
| `StkId` | typedef (from lobject.h) | Stack element pointer: `TValue*` |

## Global / File-Static State
None.

## Key Functions / Methods

### luaH_getnum
- **Signature:** `const TValue *luaH_getnum (Table *t, int key);`
- **Purpose:** Retrieve value from table by numeric (integer) key.
- **Inputs:** Table pointer, integer key.
- **Outputs/Return:** Pointer to const TValue (result), or nil if not found.
- **Side effects:** None (read-only).
- **Calls:** (defined in ltable.c)
- **Notes:** Fast path for array-like integer keys.

### luaH_setnum
- **Signature:** `TValue *luaH_setnum (lua_State *L, Table *t, int key);`
- **Purpose:** Insert/update value in table by numeric key.
- **Inputs:** Lua state (for allocation), table pointer, integer key.
- **Outputs/Return:** Pointer to TValue (destination for assignment).
- **Side effects:** May reallocate/rehash table; may trigger GC.
- **Calls:** (defined in ltable.c)
- **Notes:** May expand hash if key not already present.

### luaH_get / luaH_set
- **Signature:** `const TValue *luaH_get (Table *t, const TValue *key);` / `TValue *luaH_set (lua_State *L, Table *t, const TValue *key);`
- **Purpose:** Generic get/set for any key type (determined by TValue tag).
- **Inputs:** Table, key (as TValue with type tag).
- **Outputs/Return:** Pointer to value (const for get, mutable for set).
- **Side effects:** Set may reallocate/trigger GC.
- **Notes:** Dispatches to numeric or string path based on key type.

### luaH_getstr
- **Signature:** `const TValue *luaH_getstr (Table *t, TString *key);`
- **Purpose:** Lookup by string key (common case).
- **Inputs:** Table, TString pointer.
- **Outputs/Return:** Pointer to value or nil.
- **Notes:** Optimized for string keys.

### luaH_setstr
- **Signature:** `TValue *luaH_setstr (lua_State *L, Table *t, TString *key);`
- **Purpose:** Insert/update by string key.
- **Inputs:** Lua state, table, TString key.
- **Outputs/Return:** Mutable pointer to value slot.

### luaH_new
- **Signature:** `Table *luaH_new (lua_State *L, int narray, int lnhash);`
- **Purpose:** Create and initialize new table.
- **Inputs:** Lua state, initial array size, log2 of hash size.
- **Outputs/Return:** Pointer to new Table.
- **Side effects:** Allocates memory; registers with GC.

### luaH_resizearray
- **Signature:** `void luaH_resizearray (lua_State *L, Table *t, int nasize);`
- **Purpose:** Expand or shrink the array part of a table.
- **Inputs:** Lua state, table, new array size.
- **Side effects:** May reallocate array; may move values.

### luaH_next
- **Signature:** `int luaH_next (lua_State *L, Table *t, StkId key);`
- **Purpose:** Iterate: find next keyΓÇôvalue pair after given key.
- **Inputs:** Lua state, table, current key position.
- **Outputs/Return:** 1 if found (advances key on stack), 0 if end of table.
- **Side effects:** May modify stack to store next key.
- **Notes:** Non-deterministic order (hash iteration); used by `next()` and `pairs()`.

### luaH_getn
- **Signature:** `int luaH_getn (Table *t);`
- **Purpose:** Query table length (count of consecutive integer keys from 1).
- **Inputs:** Table pointer.
- **Outputs/Return:** Table length.
- **Notes:** Used by `#` operator.

### luaH_free
- **Signature:** `void luaH_free (lua_State *L, Table *t);`
- **Purpose:** Deallocate table and its contents.
- **Inputs:** Lua state (for memory manager), table.
- **Side effects:** Frees memory; unlinks from GC.

### luaH_mainposition (debug only)
- **Signature:** `Node *luaH_mainposition (const Table *t, const TValue *key);`
- **Purpose:** (Debug) Return hash table position where key should be stored.
- **Notes:** Exposed only if `LUA_DEBUG` defined.

### luaH_isdummy (debug only)
- **Signature:** `int luaH_isdummy (Node *n);`
- **Purpose:** (Debug) Check if node is a sentinel/dummy.

## Control Flow Notes
Tables are core runtime data structures. Functions are called during:
- **Initialization:** `luaH_new` creates tables on VM startup or `{}` creation.
- **Execution:** `luaH_get`, `luaH_set` called on every table access (`t[k]`).
- **Iteration:** `luaH_next` called by `pairs()` / `next()` loops.
- **GC/Shutdown:** `luaH_free` called when tables are unreachable or VM exits.

Array vs. hash decision is implicit: numeric keys in range `[1, sizearray]` use fast array path; others use hash.

## External Dependencies
- `lobject.h`: Defines `Table`, `Node`, `TValue`, `TString`, `TKey`, `StkId`, type macros, and GC header.
- Implicit: `lua_State` (Lua VM state, defined elsewhere).
