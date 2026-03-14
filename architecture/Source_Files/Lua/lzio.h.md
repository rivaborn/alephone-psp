# Source_Files/Lua/lzio.h

## File Purpose
Provides buffered input stream abstraction for Lua's I/O system. Enables efficient reading from various sources (files, strings, memory) through a pluggable reader callback and an internal buffer to minimize reader calls.

## Core Responsibilities
- Define buffered input stream structure (`Zio`) and callback-based reading protocol
- Provide dynamic memory buffer structure (`Mbuffer`) with resizing/allocation utilities
- Expose macros for efficient character-level reads with lazy filling
- Declare stream initialization and reading functions
- Separate public API from implementation details (marked as "Private Part")

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Zio` | struct | Buffered input stream: holds buffer position (`p`), remaining bytes (`n`), external reader callback, user data, and Lua state |
| `Mbuffer` | struct | Dynamic memory buffer: contains allocated buffer pointer, current size (`n`), and capacity (`buffsize`) |

## Global / File-Static State
None.

## Key Functions / Methods

### luaZ_init
- Signature: `void luaZ_init(lua_State *L, ZIO *z, lua_Reader reader, void *data)`
- Purpose: Initialize a buffered stream with a reader callback and user data
- Inputs: Lua state, uninitialized ZIO struct, reader callback, opaque user data pointer
- Outputs/Return: None (initializes in-place)
- Side effects: Initializes ZIO fields; reader callback will be invoked on demand
- Calls: Not visible in this file
- Notes: Reader is stored but not immediately called; first read triggers callback

### luaZ_read
- Signature: `size_t luaZ_read(ZIO* z, void* b, size_t n)`
- Purpose: Read up to `n` bytes from the stream into buffer `b`
- Inputs: ZIO stream, target buffer, byte count
- Outputs/Return: Number of bytes actually read
- Side effects: Advances stream position; may invoke reader callback
- Calls: Not visible in this file
- Notes: May read fewer bytes than requested if EOF encountered

### luaZ_openspace
- Signature: `char *luaZ_openspace(lua_State *L, Mbuffer *buff, size_t n)`
- Purpose: Ensure buffer has at least `n` bytes of space; reallocate if needed
- Inputs: Lua state, Mbuffer, required space
- Outputs/Return: Pointer to buffer start (after potential reallocation)
- Side effects: Reallocates buffer memory via `luaM_reallocvector`
- Calls: luaM_reallocvector (macro from lmem.h)
- Notes: Typical pre-condition before writing to buffer

### luaZ_lookahead
- Signature: `int luaZ_lookahead(ZIO *z)`
- Purpose: Peek at the next character without consuming it
- Inputs: ZIO stream
- Outputs/Return: Next character as int, or `EOZ` (-1) if EOF
- Side effects: May trigger reader callback
- Calls: Not visible in this file
- Notes: Used during parsing to decide control flow without consuming tokens

### luaZ_fill
- Signature: `int luaZ_fill(ZIO *z)`
- Purpose: (Private) Refill the internal buffer by invoking the reader callback
- Inputs: ZIO stream
- Outputs/Return: Next character as int, or `EOZ` (-1)
- Side effects: Updates ZIO's buffer position and remaining byte count; calls external reader
- Calls: Invoked implicitly by `zgetc()` macro when buffer is exhausted
- Notes: Implementation detail; should not be called directly by public code

## Macros & Helpers
- `zgetc(z)`: Fast character readΓÇöreturns next char if available in buffer, otherwise calls `luaZ_fill()`; inlines buffer check for performance
- `char2int(c)`: Safe cast unsigned char to int (avoids sign-extension issues)
- Buffer management macros: `luaZ_initbuffer`, `luaZ_buffer`, `luaZ_resetbuffer`, `luaZ_resizebuffer`, `luaZ_freebuffer` provide RAII-like operations

## Control Flow Notes
This module is invoked during the **load phase** of script execution (see `lua_load` in lua.h). The caller provides a `lua_Reader` callback that supplies chunks of data from the source (file, string, etc.). `Zio` buffers these chunks to reduce callback overhead, and consumers (lexer/parser) use `zgetc()` to read one character at a time. The stream abstracts the underlying data source, allowing Lua to load from files, strings, or dynamic sources transparently.

## External Dependencies
- **lua.h**: `lua_State`, `lua_Reader` callback typedef (`const char *(*)(lua_State *L, void *ud, size_t *sz)`)
- **lmem.h**: `luaM_reallocvector` macro for memory management
