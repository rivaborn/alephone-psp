# Source_Files/Lua/lparser.h

## File Purpose
Header for the Lua parser. Defines expression descriptors and compilation state structures used for parsing Lua source code and generating bytecode. Declares the main `luaY_parser` entry point.

## Core Responsibilities
- Define expression descriptor types for representing parsed Lua expressions in intermediate form
- Define `FuncState` structure to track parsing/compilation state of functions
- Define upvalue descriptors for capturing variables from enclosing scopes
- Declare the public parser function interface

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `expkind` | enum | Expression kind discriminator (15 variants: VVOID, VNIL, VTRUE, VFALSE, VK, VKNUM, VLOCAL, VUPVAL, VGLOBAL, VINDEXED, VJMP, VRELOCABLE, VNONRELOC, VCALL, VVARARG) |
| `expdesc` | struct | Expression descriptor: kind tag, union of (int info/aux pair or lua_Number), and patch lists (t, f) for jump addresses |
| `upvaldesc` | struct | Upvalue descriptor: kind byte and info byte |
| `FuncState` | struct | Function compilation state: current function prototype, constant/function tables, code position, register allocator, block chain, upvalue/local variable tracking |

## Global / File-Static State
None.

## Key Functions / Methods

### luaY_parser
- **Signature:** `Proto *luaY_parser(lua_State *L, ZIO *z, Mbuffer *buff, const char *name)`
- **Purpose:** Main entry point for parsing Lua source code into compiled function prototypes
- **Inputs:** Lua state (`L`), input stream (`z`), buffer workspace (`buff`), source name (`name`)
- **Outputs/Return:** Pointer to compiled `Proto` structure (function prototype with bytecode)
- **Side effects:** Reads from input stream, allocates memory, parses and compiles code
- **Calls:** (Implementation in lparser.c)
- **Notes:** Returns a `Proto` which contains bytecode instructions, constants, nested function prototypes, and metadata

## Control Flow Notes
This header supports single-pass bytecode generation with jump patching. The `expdesc` structure's `t` and `f` patch lists represent locations where true/false branch jumps must be resolved. `FuncState` tracks compilation state: `pc` (code position), `freereg` (free register pointer), and lists (`nk`, `np`, `nlocvars`, `nactvar`) for constants, nested functions, and local variables. Parsing a function creates a `FuncState`, expressions become `expdesc` descriptors, and code generation populates bytecode in the `Proto`.

## External Dependencies
- `llimits.h`: `lu_byte`, `LUAI_MAXUPVALUES`, `LUAI_MAXVARS` limits, cast macros
- `lobject.h`: `Proto`, `Table`, `TValue`, `lua_Number`, `Instruction`
- `lzio.h`: `ZIO` input stream, `Mbuffer` buffering
- `lua.h`: `lua_State`
- **Defined elsewhere:** `LexState` (forward declared; lexer state in lparser.c), `BlockCnt` (forward declared; block tracking in lparser.c)
