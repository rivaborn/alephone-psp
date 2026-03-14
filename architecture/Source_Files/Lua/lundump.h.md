# Source_Files/Lua/lundump.h

## File Purpose
Header for Lua bytecode serialization and deserialization. Declares the public API for loading precompiled Lua chunks from binary streams and dumping Lua function prototypes to bytecode format. Defines version and format constants for the Lua 5.1 binary format.

## Core Responsibilities
- Declare `luaU_undump` to deserialize bytecode from input streams
- Declare `luaU_dump` to serialize function prototypes to bytecode
- Declare `luaU_header` to generate standard Lua binary file headers
- Declare `luaU_print` for debugging/inspecting bytecode structure
- Define version, format, and header size constants for bytecode files
- Provide platform-independent interface for chunk I/O

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaU_undump
- Signature: `Proto* luaU_undump (lua_State* L, ZIO* Z, Mbuffer* buff, const char* name)`
- Purpose: Deserialize a precompiled Lua function prototype from a binary input stream
- Inputs: L (Lua state), Z (buffered input stream), buff (temporary memory buffer), name (debug source name for the chunk)
- Outputs/Return: Pointer to deserialized `Proto` structure
- Side effects: Reads bytes from ZIO stream, allocates memory via Lua allocator
- Calls: Defined in lundump.c
- Notes: Called during chunk loading; name parameter used for error reporting and debug info

### luaU_dump
- Signature: `int luaU_dump (lua_State* L, const Proto* f, lua_Writer w, void* data, int strip)`
- Purpose: Serialize a compiled Lua function prototype to bytecode format via a writer callback
- Inputs: L (Lua state), f (function prototype to serialize), w (output writer function), data (opaque context for writer), strip (boolean flag to omit debug information)
- Outputs/Return: 0 on success, non-zero on write error
- Side effects: Invokes writer callback multiple times, may allocate temporary memory
- Calls: Defined in ldump.c
- Notes: Writer callback invoked for sequential bytecode chunks; strip=1 removes source line info and variable names

### luaU_header
- Signature: `void luaU_header (char* h)`
- Purpose: Generate the standard Lua binary file header
- Inputs: h (output buffer, must be at least LUAC_HEADERSIZE bytes)
- Outputs/Return: None (header written to buffer)
- Side effects: Writes LUAC_HEADERSIZE bytes to provided buffer
- Calls: Defined in lundump.c
- Notes: Header contains version, format, and platform info; required at start of all bytecode files

### luaU_print
- Signature: `void luaU_print (const Proto* f, int full)`
- Purpose: Print human-readable representation of compiled bytecode structure
- Inputs: f (function prototype to inspect), full (verbosity: 0=brief, non-zero=detailed)
- Outputs/Return: None (output to stdout via Lua debug channel)
- Side effects: Writes to debug output stream
- Calls: Defined in print.c
- Notes: Only declared when `luac_c` macro is defined; used by luac (Lua compiler tool)

## Control Flow Notes
This header is part of Lua's chunk I/O subsystem. Functions are called at these lifecycle points:
- **Initialization**: `luaU_undump` deserializes precompiled chunks when loading bytecode files
- **Compilation**: `luaU_dump` serializes compiled prototypes; `luaU_header` prefixes bytecode files
- **Debugging**: `luaU_print` inspects bytecode (luac compiler only)

No internal state management; all state is passed via parameters.

## External Dependencies
- **Includes**: `lobject.h` (provides `Proto`), `lzio.h` (provides `ZIO`, `Mbuffer`)
- **Types from elsewhere**: `lua_State` (lua.h), `lua_Writer` (lua.h)
- **Implementation files**: lundump.c (undump/header), ldump.c (dump), print.c (print)

## Notes
- `LUAC_VERSION = 0x51`: Lua 5.1 version identifier
- `LUAC_FORMAT = 0`: Official bytecode format version (0 = standard)
- `LUAC_HEADERSIZE = 12`: Binary header is always 12 bytes
- `LUAI_FUNC` macro: Visibility/calling convention for exported functions
