# Source_Files/Lua/lstring.h

## File Purpose
Header file for Lua's string table management system. Declares macros and functions to create, intern, and manage strings in a centralized hash table, preventing duplicate strings in memory. Also handles userdata object allocation.

## Core Responsibilities
- String table creation and interning (all Lua strings stored centrally)
- Memory size calculations for string and userdata allocations
- Convenience macros for string creation from C strings and literals
- String marking as fixed (protected from garbage collection)
- String table resizing and management
- Userdata object allocation and lifetime

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TString` | union (defined in lobject.h) | String object with hash, length, GC header |
| `Udata` | union (defined in lobject.h) | Userdata object with metatable, environment, length |

## Global / File-Static State
None.

## Key Functions / Methods

### luaS_newlstr
- Signature: `TString *luaS_newlstr(lua_State *L, const char *str, size_t l)`
- Purpose: Create a new string of specified length in the string table
- Inputs: `L` (Lua state), `str` (char pointer), `l` (string length)
- Outputs/Return: Pointer to interned `TString` object
- Side effects: Allocates memory, updates global string table, triggers GC if needed
- Calls: (declared here, defined elsewhere)
- Notes: Foundation for both `luaS_new` and `luaS_newliteral` macros

### luaS_resize
- Signature: `void luaS_resize(lua_State *L, int newsize)`
- Purpose: Resize the string table hash table when load factor changes
- Inputs: `L` (Lua state), `newsize` (new hash table size)
- Outputs/Return: None
- Side effects: Reallocates hash table, rehashes all strings
- Calls: (declared here, defined elsewhere)
- Notes: Called by GC during sweep phase

### luaS_newudata
- Signature: `Udata *luaS_newudata(lua_State *L, size_t s, Table *e)`
- Purpose: Allocate a new userdata object with optional environment table
- Inputs: `L` (Lua state), `s` (data size), `e` (environment table pointer)
- Outputs/Return: Pointer to newly allocated `Udata` object
- Side effects: Allocates memory, links into GC list
- Calls: (declared here, defined elsewhere)
- Notes: Userdata bridges C data with Lua; `e` may be NULL

## Macro Functions
- **`luaS_new(L, s)`**: Convenience wrapperΓÇöcreates string from null-terminated C string by calling `luaS_newlstr` with `strlen`
- **`luaS_newliteral(L, s)`**: Create string from string literal with length computed at compile-time (avoids strlen)
- **`sizestring(s)`**: Returns memory size of TString including char data
- **`sizeudata(u)`**: Returns memory size of Udata including opaque data
- **`luaS_fix(s)`**: Marks string as fixed by setting `FIXEDBIT` in GC header (excluded from collection)

## Control Flow Notes
Entry point for all string and userdata allocation in Lua. Called during:
- Script parsing (string literals and identifiers)
- Runtime string operations (concatenation, function names)
- C API calls creating userdata
- String table resizing triggered by garbage collector

Integrates with GC system (`lgc.h`) for write barriers and color marking.

## External Dependencies
- **`lgc.h`**: GC constants (`FIXEDBIT`, `l_setbit` macro) and barrier operations
- **`lobject.h`**: Type definitions (`TString`, `Udata`, `GCObject`, `CommonHeader`)
- **`lstate.h`**: `lua_State` structure and global state access
