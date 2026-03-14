# Source_Files/Lua/language_definition.h
## File Purpose

A symbolic constant mapping file for game scripting language (Pfhortran), translating human-readable mnemonic names to internal numeric IDs. Serves as a data definition file for items, monsters, damage types, sounds, projectiles, and game mechanics. Supports dual naming conventions (with and without `_` prefix) for backwards compatibility.

## Core Responsibilities

- Define item type mnemonics (weapons, powerups, ammo) and their numeric codes
- Define monster/enemy type mnemonics and classifications
- Define damage type constants for tracking source of harm
- Define player and monster action/mode states
- Define visual effect (fader) types for cinematics and screen transitions
- Define audio/sound event IDs for gameplay events
- Define projectile types and their associated numeric IDs
- Define polygon/level geometry type constants
- Provide dual naming interface for script compatibility

## External Dependencies

- **Undefined symbolic constants** (referenced but not defined in this file):
  - `_panel_is_oxygen_refuel`, `_panel_is_shield_refuel`, `_panel_is_double_shield_refuel`, `_panel_is_triple_shield_refuel`
  - `_weapon_fist`, `_weapon_pistol`, `_weapon_plasma_pistol`, `_weapon_shotgun`, `_weapon_assault_rifle`, `_weapon_smg`, `_weapon_flamethrower`, `_weapon_missile_launcher`, `_weapon_alien_shotgun`, `_weapon_ball`
  - `_game_of_most_points`, `_game_of_most_time`, `_game_of_least_points`, `_game_of_least_time`
  - These are defined elsewhere and referenced here as numeric values for script constants.

- **Language/system**: Pfhortran (game scripting language, based on file header comments); used for Lua-based or similar scripting system
- **Backwards compatibility note**: File preserves old naming schemes (underscore-prefixed) alongside cleaner modern names to avoid breaking existing scripts

# Source_Files/Lua/lapi.h
## File Purpose
Header file declaring auxiliary API functions for Lua's internal object manipulation. Provides a single entry point to push Lua values onto the interpreter stack, bridging internal object representation with stack-based API operations.

## Core Responsibilities
- Declare auxiliary API functions for object stack operations
- Provide minimal interface between Lua's internal TValue representation and public stack APIs

## External Dependencies
- **Includes:** `"lobject.h"` ΓÇö provides `TValue`, `lua_State` definitions and type system
- **Macros used:** `LUAI_FUNC` (from `llimits.h`, included transitively) ΓÇö marks internal function declarations

# Source_Files/Lua/lauxlib.h
## File Purpose
Header file declaring the Lua Auxiliary Library API for C code building Lua libraries and extensions. Provides registration, type checking, error handling, and utility functions for integrating C code with Lua 5.1.

## Core Responsibilities
- Declare library registration and initialization functions
- Provide type checking and argument validation helpers
- Declare error reporting and stack manipulation functions
- Declare string/number/integer conversion and validation functions
- Provide buffer management for efficient string concatenation
- Declare reference management (garbage collection protection)
- Declare code loading functions (from files, strings, buffers)
- Provide compatibility macros for different Lua versions

## External Dependencies
- Includes: `<stddef.h>`, `<stdio.h>`, `lua.h`
- Depends on: `lua_State`, `lua_CFunction`, `lua_Number`, `lua_Integer`, `LUALIB_API`, `LUA_REGISTRYINDEX`, `lua_pcall`, `lua_error` (from `lua.h`)
- Defines compatibility wrappers for `LUA_COMPAT_GETN`, `LUA_COMPAT_OPENLIB` based on configuration

# Source_Files/Lua/lcode.h
## File Purpose
Header for Lua's code generator. Declares the interface for emitting bytecode instructions, managing register allocation, and handling jump patching during compilation. Defines operator enums and macros for instruction field manipulation.

## Core Responsibilities
- Declare binary and unary operator enums for expression codegen dispatch
- Define macros for packing/unpacking instruction fields (ABC, ABx, AsBx formats)
- Declare functions for emitting instructions in all VM formats
- Provide expression descriptor conversion functions (to registers, constants, RK indices)
- Manage jump target patching and control flow code generation
- Handle constant table management (string/number deduplication)
- Coordinate operator code generation (prefix, infix, postfix)

## External Dependencies
- **llex.h**: Lexer types (Token, LexState) ΓÇö related but not directly used in this header
- **lobject.h**: Core object types (Proto, TString, lua_Number, TValue, Table)
- **lopcodes.h**: Instruction format, OpCode enum, instruction field macros (GET_OPCODE, SETARG_*, CREATE_ABC, etc.), constants (MAXARG_Bx, NO_REG)
- **lparser.h**: expdesc, expkind, FuncState, upvaldesc
- **llimits.h** (indirectly): Platform macros (cast, LUAI_FUNC, LUAI_DATA)

# Source_Files/Lua/ldebug.h
## File Purpose
Header file for Lua's debug interface module. Declares error reporting and code validation functions used throughout the interpreter, along with utility macros for managing program counters and debug hooks.

## Core Responsibilities
- Declare type error, concatenation error, arithmetic error, and runtime error reporting functions
- Provide program counter manipulation macros (`pcRel`)
- Provide line number lookup macro for debugging (`getline`)
- Declare code and instruction validation functions (`luaG_checkcode`, `luaG_checkopenop`)
- Declare debug hook management via `resethookcount` macro

## External Dependencies
- **Includes**: `lstate.h` (Lua state structures, call stack info)
- **External symbols used**: `lua_State`, `TValue`, `StkId`, `Proto`, `Instruction` (all defined in included headers and `lobject.h`)
- **Macro uses**: `LUAI_FUNC` (visibility/calling convention marker for Lua API)

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

## External Dependencies
- **lobject.h**: `TValue`, `StkId` (stack value type and stack index)
- **lstate.h**: `lua_State`, `CallInfo`, `global_State` (execution state)
- **lzio.h**: `ZIO` (buffered input for parser)

# Source_Files/Lua/lfunc.h
## File Purpose
Header declaring the Lua C API for function prototype and closure manipulation. Provides functions to create, inspect, and manage Lua function objects (both C closures wrapping native functions and Lua closures wrapping script functions).

## Core Responsibilities
- Create and allocate function prototypes (`Proto`)
- Allocate C closures (native function wrappers) and Lua closures (script function instances)
- Manage upvalues: create, find, and close them across function scopes
- Deallocate function objects and their upvalues during garbage collection
- Query function metadata (e.g., local variable names at debug locations)
- Provide sizing macros for closure memory allocation

## External Dependencies
- **Includes:** `"lobject.h"` ΓÇö defines `Proto`, `Closure`, `UpVal`, `TValue`, `Table`, `StkId`, and all base object structures.
- **Macros used:** `LUAI_FUNC` (marks functions exported by the Lua internals), `cast()` (casts used in size macros).
- **Defined elsewhere:** `lua_State` (Lua VM state), `StkId` (stack index typedef from `lobject.h`).

# Source_Files/Lua/lgc.h
## File Purpose
Garbage collector interface header for Lua's incremental tri-color marking GC. Defines GC state constants, bit manipulation utilities for object marking, tri-color classification macros, write barriers to preserve GC invariants, and function declarations for GC operations.

## Core Responsibilities
- Define GC finite-state machine states (pause, propagate, sweep string, sweep, finalize)
- Provide bit-level utilities for manipulating and testing the `marked` field of collectable objects
- Implement tri-color marking scheme: two white colors (for generational collection), black, and implicit gray
- Supply write barrier macros (`luaC_barrier`, `luaC_barriert`, etc.) to detect and handle blackΓåÆwhite references that violate GC invariants
- Declare entry-point functions for step-wise GC execution, full GC cycles, object linking, and finalization
- Expose thread-safe GC threshold checks for incremental collection triggering

## External Dependencies
- `#include "lobject.h"` ΓÇö Provides `GCObject`, `GCheader`, `Table`, `UpVal`, `lua_State` (via transitive includes)
- Macro helpers: `cast()`, `check_exp()` (from llimits.h via lobject.h)
- Type aliases: `lu_byte`, `size_t` (from llimits.h or standard headers)

# Source_Files/Lua/llex.h
## File Purpose

Lexical analyzer header for Lua. Defines token types, the LexState structure that maintains tokenization state, and the public API for lexing source code into a stream of tokens consumed by the parser.

## Core Responsibilities

- Define all token types (reserved keywords, operators, literals, special tokens)
- Provide the LexState structure to track position, current/lookahead tokens, and input streams
- Define SemInfo union to store semantic data for tokens (numeric constants or interned strings)
- Declare the lexer initialization and tokenization API (luaX_next, lookahead, error reporting)
- Manage string interning via luaX_newstring to deduplicate string values

## External Dependencies

- **lobject.h**: TString (interned strings), lua_Number, basic Lua value types
- **lzio.h**: ZIO (buffered input stream), Mbuffer (dynamic buffer)
- **lua.h**: lua_State (main Lua context, included indirectly)
- **luaX_tokens[]**: Token name table, defined in llex.c (implementation file)


# Source_Files/Lua/llimits.h
## File Purpose

Core infrastructure header for Lua that defines installation-dependent limits, type aliases, and utility macros. Establishes portable integer types, memory bounds, VM instruction encoding, and assertion/debugging infrastructure used throughout the Lua engine.

## Core Responsibilities

- Define portable integer types (lu_int32, lu_mem, l_mem, lu_byte) abstracted via configuration
- Establish hard limits for stack depth, string tables, and memory allocation
- Provide type-casting macros and assertion helpers for safety and debugging
- Define the Instruction type (VM instruction encoding, 32-bit unsigned)
- Configure thread-safety primitives (lua_lock, lua_unlock, threadyield)
- Provide pointer-to-integer hashing conversion (IntPoint)
- Define stack and buffer size thresholds (MAXSTACK, MINSTRTABSIZE, LUA_MINBUFFER)

## External Dependencies

- **Standard library**: `<limits.h>` (INT_MAX), `<stddef.h>` (size_t, NULL)
- **Lua headers**: `"lua.h"` (version, type constants, lua_State, function pointers)
- **Config abstraction**: References macros from `luaconf.h` (included transitively via lua.h): LUAI_UINT32, LUAI_UMEM, LUAI_MEM, LUAI_USER_ALIGNMENT_T, LUAI_UACNUMBER
- **Defined elsewhere**: lua_State (opaque in this file, declared in lua.h)

# Source_Files/Lua/lmem.h
## File Purpose
Memory manager interface header for Lua 5.1. Provides a macro-based abstraction over allocation, deallocation, and reallocation operations with built-in overflow checking and a unified error path through `lua_State`.

## Core Responsibilities
- Define allocation macros for single objects (`luaM_new`), arrays (`luaM_newvector`), and dynamic growth (`luaM_growvector`)
- Define deallocation macros for flexible cleanup patterns (`luaM_free`, `luaM_freemem`, `luaM_freearray`)
- Protect against integer overflow when computing allocation sizes via `luaM_reallocv` size checks
- Provide a standard error message constant for memory exhaustion
- Route all memory operations through a central `lua_State` context for error propagation

## External Dependencies
- **Includes:**
  - `<stddef.h>` ΓÇô `size_t` type
  - `"llimits.h"` ΓÇô `MAX_SIZET`, `cast` macro
  - `"lua.h"` ΓÇô `lua_State` struct, error constants
- **Defined elsewhere:**
  - `lua_State`: Interpreter state context
  - `MAX_SIZET`: Safe upper bound for size_t (defined in `llimits.h`)

# Source_Files/Lua/lobject.h
## File Purpose
Defines the core type system for Lua objects, including the tagged value representation (`TValue`) that holds all Lua values, and data structures for collectable objects (strings, tables, functions, closures, upvalues). This is the foundation for type representation throughout the Lua runtime.

## Core Responsibilities
- Define the union-based tagged value system (`Value` + type tag) for representing all Lua types
- Provide macros for runtime type checking and safe value access
- Define structures for all collectable garbage-collected objects (strings, tables, functions, closures, upvalues, userdata)
- Define function prototypes (`Proto`) with metadata for code, constants, local variables, and upvalues
- Define hash table implementation (`Table`, `Node`, `TKey`) with array and hash components
- Export utility functions for type conversion and object comparison

## External Dependencies
- `<stdarg.h>` ΓÇô variadic argument handling
- `llimits.h` ΓÇô type limits, alignment helpers, casting macros
- `lua.h` ΓÇô public API type definitions, Lua state, type constants (LUA_TNIL, LUA_TNUMBER, etc.)
- `luaconf.h` (included indirectly via lua.h) ΓÇô configuration and platform-specific types

**Defined elsewhere:**
- `luaO_log2`, `luaO_int2fb`, `luaO_fb2int`, `luaO_rawequalObj`, `luaO_str2d`, `luaO_pushvfstring`, `luaO_pushfstring`, `luaO_chunkid` ΓÇô implemented in ldo.c or lopcodes.c
- `struct Table`, `struct Proto`, `lua_State` ΓÇô used but struct members defined here; lua_State defined in lstate.h

# Source_Files/Lua/lopcodes.h
## File Purpose
Defines the instruction format, field layout, and complete opcode enumeration for the Lua virtual machine. Provides macros for encoding/decoding instructions and querying opcode properties. This is the core bytecode definition header used by the Lua compiler and interpreter.

## Core Responsibilities
- Specify instruction bit layout: 6-bit opcode, 8-bit A field, 9-bit B and C fields (or 18-bit Bx combined)
- Provide macros to extract and modify individual instruction fields (GET_OPCODE, SETARG_A, etc.)
- Define all 38 VM opcodes (OP_MOVE, OP_LOADK, OP_CALL, OP_RETURN, etc.)
- Handle register/constant (RK) indexing with sign-bit discrimination (BITRK)
- Declare external metadata tables (opcode modes and names)
- Define instruction creation macros (CREATE_ABC, CREATE_ABx)

## External Dependencies
- `#include "llimits.h"` ΓÇö defines `Instruction` type (lu_int32), `lu_byte`, `cast()` macro, `MAX_INT`, `LUAI_BITSINT`, and `LUAI_DATA` annotation
- References to `lua.h` in copyright notice (not included here)
- External data `luaP_opmodes[NUM_OPCODES]` and `luaP_opnames[NUM_OPCODES+1]` ΓÇö defined elsewhere (likely lopcodes.c)

# Source_Files/Lua/lparser.h
## File Purpose
Header for the Lua parser. Defines expression descriptors and compilation state structures used for parsing Lua source code and generating bytecode. Declares the main `luaY_parser` entry point.

## Core Responsibilities
- Define expression descriptor types for representing parsed Lua expressions in intermediate form
- Define `FuncState` structure to track parsing/compilation state of functions
- Define upvalue descriptors for capturing variables from enclosing scopes
- Declare the public parser function interface

## External Dependencies
- `llimits.h`: `lu_byte`, `LUAI_MAXUPVALUES`, `LUAI_MAXVARS` limits, cast macros
- `lobject.h`: `Proto`, `Table`, `TValue`, `lua_Number`, `Instruction`
- `lzio.h`: `ZIO` input stream, `Mbuffer` buffering
- `lua.h`: `lua_State`
- **Defined elsewhere:** `LexState` (forward declared; lexer state in lparser.c), `BlockCnt` (forward declared; block tracking in lparser.c)

# Source_Files/Lua/lstate.h
## File Purpose
Core header defining Lua's execution state structures. Declares the global state shared across all threads, per-thread execution state, call stack frames, and provides type-conversion macros for garbage-collectable objects. Manages thread creation and destruction.

## Core Responsibilities
- Define `global_State` structure for VM-wide shared state (memory allocation, GC, string interning, registry)
- Define `lua_State` structure for per-thread execution context (stack, call info, hooks, upvalues)
- Define `CallInfo` for tracking function call frames on the call stack
- Define `stringtable` hash table for string interning and deduplication
- Provide accessor macros (`G()`, `gt()`, `registry()`, `curr_func()`) for navigating state
- Define `GCObject` union encompassing all garbage-collectable types
- Provide type-casting macros (`gco2ts()`, `gco2h()`, `gco2cl()`, etc.) for safe GCObject conversion

## External Dependencies
- **Includes:** `lua.h` (Lua version and type definitions), `lobject.h` (TValue, GCheader, Closure, Proto, Table, UpVal, TString, Udata), `ltm.h` (tag-method enumeration), `lzio.h` (Mbuffer)
- **Defined elsewhere:** `struct lua_longjmp` (ldo.c), conversion functions and GC internals

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

## External Dependencies
- **`lgc.h`**: GC constants (`FIXEDBIT`, `l_setbit` macro) and barrier operations
- **`lobject.h`**: Type definitions (`TString`, `Udata`, `GCObject`, `CommonHeader`)
- **`lstate.h`**: `lua_State` structure and global state access

# Source_Files/Lua/ltable.h
## File Purpose
Public interface header for Lua's hash table implementation. Declares functions for table creation, value lookup/insertion, and iteration. Provides utility macros for accessing hash table node structures.

## Core Responsibilities
- Define accessor macros for hash table nodes (`gnode`, `gkey`, `gval`, `gnext`)
- Declare public table API functions (create, get, set, resize, iterate)
- Support multiple key types: numeric, string, and generic `TValue` keys
- Provide table size queries and free operations
- Expose debug utilities for internal table structure inspection

## External Dependencies
- `lobject.h`: Defines `Table`, `Node`, `TValue`, `TString`, `TKey`, `StkId`, type macros, and GC header.
- Implicit: `lua_State` (Lua VM state, defined elsewhere).

# Source_Files/Lua/ltm.h
## File Purpose
Defines Lua's tag method (metamethod) system, which implements operator overloading and special behaviors for Lua objects. Declares the enumeration of all tag method types and provides macros and functions for efficient tag method lookup during runtime.

## Core Responsibilities
- Define the `TMS` enum cataloging all supported tag methods (INDEX, NEWINDEX, GC, arithmetic ops, etc.)
- Provide optimized macros (`gfasttm`, `fasttm`) for fast tag method lookups with caching via flags
- Declare functions for tag method retrieval from tables and objects
- Initialize the tag method subsystem at Lua startup
- Separate fast-path methods (INDEX, NEWINDEX, GC, MODE, EQ) from regular methods for performance

## External Dependencies
- `lobject.h`: defines `TValue`, `Table`, `TString`, and related macros
- Lua core state structures (`lua_State`, accessed via `G(l)` macro)
- Tag method declarations implemented elsewhere (likely `ltm.c`)

# Source_Files/Lua/lua.h
## File Purpose
Public C API header for Lua 5.1.2, an extensible extension language. Defines the complete interface for embedding Lua in C/C++ applications, enabling stack-based value access, function calls, coroutine management, and debugging hooks.

## Core Responsibilities
- Define opaque `lua_State` type and function pointer signatures (`lua_CFunction`, `lua_Reader`, `lua_Writer`, `lua_Alloc`, `lua_Hook`)
- Declare Lua type constants (`LUA_TNIL`, `LUA_TBOOLEAN`, `LUA_TNUMBER`, etc.) and error codes
- Provide stack manipulation functions for transferring data between Lua and C
- Enable C code to call Lua functions and vice versa
- Define table/userdata access functions for object manipulation
- Support coroutines, garbage collection control, and debugging hooks
- Supply utility macros for common operations (type checks, stack operations)

## External Dependencies
- `<stdarg.h>` ΓÇö for `va_list` in `lua_pushvfstring()`
- `<stddef.h>` ΓÇö for `size_t`
- `"luaconf.h"` ΓÇö configuration defines (`LUA_NUMBER`, `LUA_INTEGER`, `LUA_API`, type constants, memory limits)

# Source_Files/Lua/lua_map.cpp
## File Purpose
Implements Lua C bindings for Marathon/Aleph One game world map data. Registers map geometry (polygons, lines, endpoints, sides), dynamic entities (platforms, lights, media/liquids), and map metadata (fog, level properties, annotations) as Lua userdata with getter/setter functions, enabling Lua scripts to query and modify the game world at runtime.

## Core Responsibilities
- Register all map-related Lua classes (geometry, platforms, lights, sides, control panels, media) with Lua interpreter
- Implement property accessors (getters/setters) for map objects and their attributes
- Bridge C++ game world state with Lua scripting environment
- Provide container-style access to iterate over game world elements (lines, polygons, platforms, lights, etc.)
- Load backwards-compatibility Lua functions for deprecated APIs
- Handle type conversions between Lua and C++ (world coordinates, bitmasks, enum values)

## External Dependencies

**Lua C API:**
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô stack manipulation, error handling, registry access

**Game World Data:**
- `map.h` ΓÇô geometry structures (`polygon_data`, `line_data`, `endpoint_data`, `side_data`, `object_data`), global lists (`PolygonList`, `LineList`, etc.), and accessor macros
- `lightsource.h` ΓÇô light structures and accessors (`LightList`, `get_light_data()`)
- `media.h` ΓÇô media/liquid structures (`MediaList`, `get_media_data()`)
- `platforms.h` ΓÇô platform structures and state functions (`PlatformList`, `get_platform_data()`, `set_platform_state()`)
- `collection_definition.h` ΓÇô collection metadata (forward declared, `get_collection_definition()`)

**Rendering/Engine:**
- `OGL_Setup.h` ΓÇô fog data retrieval (`OGL_GetFogData()`, `OGL_Fog_AboveLiquid`, `OGL_Fog_BelowLiquid`)
- `SoundManager.h` ΓÇô audio system
- `lua_monsters.h`, `lua_objects.h`, `lua_templates.h` ΓÇô other Lua binding modules

**External State/Functions (defined elsewhere, called here):**
- Global mutable state: `static_world`, `dynamic_world`, `MapAnnotationList`, various `*List` containers
- Functions: `get_light_status()`, `set_light_status()`, `set_tagged_light_statuses()`, `try_and_change_tagged_platform_states()`, `assume_correct_switch_position()`, `adjust_platform_for_media()`, `recalculate_redundant_*_data()`, `new_side()`, `number_of_terminal_texts()`
- Macros: `PLATFORM_IS_ACTIVE()`, `LINE_IS_SOLID()`, `GET_DESCRIPTOR_SHAPE()`, `BUILD_DESCRIPTOR()`, etc.

# Source_Files/Lua/lua_map.h
## File Purpose
Declares Lua-C bindings for map geometry and related elements in the Aleph One engine. Exposes polygons, lines, sides, platforms, lights, tags, terminals, and media to Lua scripts via template-based wrapper classes.

## Core Responsibilities
- Declare Lua-C wrapper typedefs for map structural elements (lines, polygons, sides)
- Declare enum wrapper classes for map-related enumerations (damage types, transfer modes, control panel types, etc.)
- Declare container classes for iterating collections of map elements
- Export the registration function to bind map API to a Lua interpreter

## External Dependencies
- **Lua headers:** `lua.h`, `lauxlib.h`, `lualib.h` (Lua 5.1 API)
- **Game headers:** `map.h` (map data structures), `lightsource.h` (light definitions)
- **Template library:** `lua_templates.h` (L_Class, L_Enum, L_Container, L_EnumContainer template definitions)
- **Common headers:** `cseries.h` (cross-platform utilities)

# Source_Files/Lua/lua_mnemonics.h
## File Purpose
Defines constant lookup tables that map human-readable string identifiers (mnemonics) to integer codes for Lua script integration. Enables game content to reference entities, damage types, effects, sounds, and other game constants by name rather than magic numbers.

## Core Responsibilities
- Provide string-to-integer mappings for 23 distinct game concept categories
- Support Lua scripting access to engine-level constants
- Define sentinel-terminated lookup arrays for efficient runtime lookups
- Centralize mnemonic definitions for game entities (monsters, items, weapons, effects) and mechanics (difficulty, game modes, control panels)

## External Dependencies
- **Type `int32`**: Custom typedef or platform header; defines mnemonic values
- **Symbolic constants** (e.g., `_game_of_most_points` in `Lua_ScoringMode_Mnemonics`): Defined elsewhere, likely in a game constants header
- **Presumed lookup infrastructure**: Some other translation unit must iterate/search these arrays to bind Lua table keys to C values


# Source_Files/Lua/lua_monsters.cpp
## File Purpose
Implements Lua scripting bindings for monsters in the game engine. Allows Lua scripts to create monsters, query/modify their properties (position, health, mode, action), and trigger actions (damage, pathfinding, sound effects).

## Core Responsibilities
- Register Lua monster classes (`Lua_Monster`, `Lua_MonsterType`, `Lua_MonsterClass`) with the Lua state
- Expose monster instance methods (attack, damage, move_by_path, play_sound, position)
- Provide property accessors for monster types (class, enemies, friends, immunities, weaknesses, etc.)
- Provide property accessors for monster instances (vitality, facing, mode, action, polygon, visibility)
- Implement factory method for creating new monsters
- Provide backward-compatibility layer of wrapper functions for old Lua API

## External Dependencies
- **Lua C API:** lua.h, lauxlib.h, lualib.h (luaL_reg, lua_State, luaL_error, etc.)
- **Template classes:** lua_templates.h (L_Class, L_Enum, L_Container, L_EnumContainer)
- **Engine internals:** 
  - monsters.h (monster_data, monster_definition, MAXIMUM_MONSTERS_PER_MAP, activation/movement/damage functions)
  - flood_map.h (pathfinding: new_path, delete_path, move_along_path)
  - player.h (monster_index_to_player_index)
  - lua_map.h (Lua_Polygon, Lua_DamageType, etc.)
  - lua_objects.h (Lua_EffectType, Lua_ItemType, Lua_Sound)
  - lua_player.h (Lua_Player)
  - boost/bind.hpp (boost::bind for Length callback)
- **External symbols used:** get_monster_data, get_monster_definition_external, get_object_data, change_monster_target, activate_monster, deactivate_monster, damage_monster, ::new_monster, play_object_sound, advance_monster_path, set_monster_action, set_monster_mode, monster_pathfinding_cost_function, get_polygon_data, GetMemberWithBounds, remove_object_from_polygon_object_list, add_object_to_polygon_object_list, monster_index_to_player_index

# Source_Files/Lua/lua_monsters.h
## File Purpose
Defines the Lua scripting interface for monsters in the game engine. Provides type aliases that wrap C++ monster data structures as Lua-accessible classes and containers, allowing Lua scripts to query and interact with game monsters.

## Core Responsibilities
- Define Lua bindings for individual monster instances (`Lua_Monster`)
- Define Lua bindings for the monsters collection (`Lua_Monsters`)
- Define Lua bindings for monster action enumerations (`Lua_MonsterAction`)
- Register all monster-related Lua types and functions with the Lua runtime
- Serve as the public interface between C++ monster management and Lua scripting

## External Dependencies
- **Notable includes:**
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua C API (wrapped in `extern "C"`)
  - `map.h` ΓÇô Game world/map structures (monster location context)
  - `monsters.h` ΓÇô Engine's C++ monster data structures and functions
  - `lua_templates.h` ΓÇô Template classes for Lua bindings (`L_Class`, `L_Container`, `L_Enum`)
- **External symbols used but not defined here:**
  - Template implementations from `lua_templates.h` (L_Class, L_Container, L_Enum)
  - Monster definitions and game world state from `monsters.h` and `map.h`
  - Lua C API functions (lua_State, luaL_Reg, etc.)

# Source_Files/Lua/lua_objects.cpp
## File Purpose
Implements Lua bindings for game map objects (Effects, Items, Scenery). Exposes object creation, deletion, and property access to Lua scripts via template-based wrappers around C API calls. Provides constructors, getters/setters, and registration with the Lua VM.

## Core Responsibilities
- Register object classes (Effects, Items, Scenery) with Lua metatable system
- Implement object constructors (`Effects.new()`, `Items.new()`, `Scenery.new()`)
- Expose object properties (position, facing, polygon, type) as Lua getter/setter functions
- Handle object deletion with cleanup (e.g., deanimation for scenery)
- Provide object sound playback interface
- Validate object indices and enforce state constraints
- Maintain backwards-compatibility wrapper functions for legacy Lua scripts

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h`
- **Game Engine Objects**: `effects.h`, `items.h`, `monsters.h`, `scenery.h`, `player.h` (for object_data access)
- **Map/World**: `map.h` (from lua_map.h), `scenery_definitions.h` (for scenery type mnemonics)
- **Templates/Bindings**: `lua_templates.h` (L_Class, L_Container, L_Enum base classes), `lua_map.h` (Lua_Polygon class)
- **Sound**: `SoundManager.h`
- **Boost**: `boost/bind.hpp` (for dynamic limit callbacks)
- **Defined Elsewhere**: `new_effect()`, `new_item()`, `new_scenery()`, `remove_map_object()`, `damage_scenery()`, `deanimate_scenery()`, `randomize_scenery_shape()`, `play_object_sound()`, `get_dynamic_limit()`, object owner enum constants

# Source_Files/Lua/lua_objects.h
## File Purpose
Declares Lua language bindings for game map objects (effects, items, scenery, sounds) in Aleph One (Marathon engine port). Uses template-based wrappers to expose C++ game entities as Lua objects with introspection and iteration support.

## Core Responsibilities
- Define typedefs for Lua class and container wrappers for game object types
- Export external string constants used as Lua metatable identifiers
- Declare the registration function that initializes all object bindings in a Lua state
- Provide separate class and container interfaces for singular objects and object collections
- Support both regular classes and enum-like types with mnemonic string lookups

## External Dependencies
- **Lua 5.1 API:** lua.h, lauxlib.h, lualib.h ΓÇö Lua interpreter and auxiliary library functions
- **Game world:** items.h (item type definitions), map.h (map/world geometry structures) 
- **Lua binding templates:** lua_templates.h (provides `L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`, `L_LazyEnum`)
- **Platform utilities:** cseries.h (core series library with platform abstractions)

# Source_Files/Lua/lua_player.cpp
## File Purpose
Implements Lua script bindings for player-related gameplay systems in Aleph One (Marathon game engine). Exposes player data (position, velocity, items, weapons), action input, cameras, HUD overlays, compass, and global game/music state to Lua scripts. Also provides ~50 backward-compatibility wrapper functions for legacy script APIs.

## Core Responsibilities
- Bind player state (position, velocity, energy, weapons, kills) to Lua with type-safe getters/setters
- Manage action flags (input state) from Lua during idle phase via action queues
- Control camera systems including path-based cinematics with spatial waypoints and orientation keyframes
- Manage player HUD overlays (icons, text, colors) for script-driven UI
- Expose compass/beacon system with directional state and world coordinates
- Control game settings (difficulty, game type, scoring mode, time limits)
- Manage music playback, fading, and validation
- Register all player-related Lua classes with the interpreter at startup

## External Dependencies

Notable includes: ActionQueues.h, lua_templates.h, lua_script.h, map.h, player.h, Music.h, Crosshairs.h, fades.h, game_window.h, interface.h, SoundManager.h, ViewControl.h, Random.h

Key external functions: get_player_data(), try_and_add_player_item(), destroy_players_ball(), select_next_best_weapon(), GetGameQueue(), Crosshairs_IsActive/SetActive(), SetScriptHUDIcon/Text/Color/Square(), Music::instance(), instantiate_physics_variables(), mark_shield_display_as_dirty(), draw_panels()

# Source_Files/Lua/lua_player.h
## File Purpose
Lua C binding header exposing game player objects to Lua scripts. Provides typed wrappers for individual players and a container for accessing all players via the Lua state. Part of Aleph One's scripting subsystem, conditionally compiled when `HAVE_LUA` is defined.

## Core Responsibilities
- Define `Lua_Player` class template specialization wrapping individual player indices
- Define `Lua_Players` container template for iterating/accessing players from Lua
- Declare registration function to bind player functionality to a Lua state
- Bridge C++ game world (via `map.h`) with Lua script environment

## External Dependencies
- **Lua C API**: `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.1.2 virtual machine interface
- **Template infrastructure**: `lua_templates.h` ΓÇö `L_Class<>` and `L_Container<>` templates for binding patterns
- **Game world**: `map.h` ΓÇö provides static/dynamic world data; player indices likely reference entities in this system
- **Common series**: `cseries.h` ΓÇö compiler/platform compatibility layer, SDL dependencies
- **Defined elsewhere**: `Lua_Player_register()` implementation; actual player data structures referenced by index

# Source_Files/Lua/lua_projectiles.cpp
## File Purpose
Implements Lua bindings for the projectile system in the Aleph One game engine (Marathon). Exposes projectile properties, creation, and management to Lua scripts through a template-based class wrapping system.

## Core Responsibilities
- Expose projectile properties (position, orientation, damage, owner, target) as Lua accessors
- Convert between game engine units (fixed-point angles, world units) and Lua-friendly formats (degrees, floating-point coordinates)
- Provide Lua methods to create new projectiles and manipulate their state
- Register Lua classes for projectile types and associated damage definitions
- Define backward-compatibility layer for legacy Lua scripts
- Manage validity checks for projectile indices

## External Dependencies
- **Notable includes:**
  - `lua.h`, `lauxlib.h`, `lualib.h` (Lua C API)
  - `lua_templates.h` (template-based Lua binding infrastructure: L_Class, L_Container, L_Enum)
  - `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h` (other Lua bindings)
  - `dynamic_limits.h` (get_dynamic_limit)
  - `map.h`, `monsters.h`, `player.h`, `projectiles.h` (core engine data structures and functions)
  - `boost/bind.hpp` (boost::bind for callback binding)

- **Defined elsewhere:**
  - `get_projectile_data()`, `::new_projectile()` ΓÇö projectile management
  - `get_object_data()`, `play_object_sound()`, `remove_object_from_polygon_object_list()`, `add_object_to_polygon_object_list()` ΓÇö object/polygon management
  - `Lua_Monster::Push()`, `Lua_Player::Is()`, `Lua_Polygon::Push()`, `Lua_Sound::ToIndex()`, `Lua_ProjectileType::ToIndex()`, `Lua_DamageType::Push()` ΓÇö other Lua class bindings
  - `SLOT_IS_USED()` macro ΓÇö slot validity check
  - `WORLD_ONE`, `FIXED_ONE`, `FULL_CIRCLE` constants ΓÇö unit conversion factors
  - `projectile_definitions.h` (included with DONT_REPEAT_DEFINITIONS macro) ΓÇö projectile type definitions

# Source_Files/Lua/lua_projectiles.h
## File Purpose
Header file defining Lua bindings for the projectiles subsystem. Declares type-alias wrappers around template classes to expose projectile objects and projectile types to Lua scripts. Only compiled when `HAVE_LUA` is defined.

## Core Responsibilities
- Declares Lua class wrapper (`Lua_Projectile`) for individual projectile game objects
- Declares Lua container wrapper (`Lua_Projectiles`) for projectile collections
- Declares Lua enum wrapper (`Lua_ProjectileType`) for projectile type enumerations
- Declares Lua enum container wrapper (`Lua_ProjectileTypes`) for projectile type collections
- Declares the module registration function to initialize Lua bindings
- Provides type-safe bridging between game engine and Lua script layer

## External Dependencies
- **Bundled headers:**
  - `cseries.h` ΓÇô project compatibility/foundation layer
  - `lua.h`, `lauxlib.h`, `lualib.h` ΓÇô Lua 5.1 C API (conditional on `HAVE_LUA`)
  - `lua_templates.h` ΓÇô template classes (`L_Class`, `L_Container`, `L_Enum`, `L_EnumContainer`) that power all bindings

# Source_Files/Lua/lua_script.cpp
## File Purpose
Lua script integration layer for a Marathon-like game engine (Aleph One). Manages loading, execution, and cleanup of Lua scripts; registers C++ functions and game constants for Lua; and invokes Lua event callbacks throughout the game lifecycle.

## Core Responsibilities
- Load Lua scripts from buffers and execute them within a Lua state
- Register native C functions exposing game engine capabilities to Lua
- Declare game constants (item types, object counts, damage types, etc.) as Lua globals
- Invoke Lua event triggers at key game moments (player damage, item pickup, switch activation, etc.)
- Provide Lua with direct player control, camera manipulation, and game state queries
- Invalidate Lua object references when game entities (monsters, projectiles, items) are destroyed
- Manage Lua-controlled HUD display, screen fades, and gameplay UI
- Toggle script muting and handle script lifecycle cleanup

## External Dependencies
**Lua Runtime:**
- `lua.h`, `lauxlib.h`, `lualib.h` ΓÇö Lua 5.1 interpreter and auxiliary library
- Standard libs: base, table, string, math, debug (IO and loadlib omitted for security)

**Game Engine:**
- `player.h`, `monsters.h`, `projectiles.h`, `items.h`, `world.h` ΓÇö Game object definitions
- `render.h`, `screen.h`, `vbl.h` ΓÇö Rendering and frame timing
- `shell.h`, `interface.h` ΓÇö Game state and UI
- `network.h`, `network_games.h` ΓÇö Multiplayer support
- `lightsource.h`, `platforms.h`, `computer_interface.h` ΓÇö Map features
- `physics_models.h`, `Random.h` ΓÇö Physics and randomness
- `fades.h`, `Music.h`, `SoundManager.h` ΓÇö Audio/visual effects
- `Console.h` ΓÇö Debug console
- `Crosshairs.h`, `ViewControl.h`, `OGL_Setup.h` ΓÇö Camera and rendering config

**Lua Binding Modules (defined elsewhere):**
- `lua_map.h`, `lua_monsters.h`, `lua_objects.h`, `lua_player.h`, `lua_projectiles.h` ΓÇö Type/function wrappers for game classes

**Constants:**
- `language_definition.h` ΓÇö Game constants (included with `DONT_REPEAT_DEFINITIONS`)
- `item_definitions.h`, `monster_definitions.h` ΓÇö Object metadata

# Source_Files/Lua/lua_script.h
## File Purpose
Declares the C/C++ interface for Lua script integration in a game engine (Aleph One). Provides event callbacks, script lifecycle management, and game state queries to allow Lua scripts to respond to and influence gameplay.

## Core Responsibilities
- **Lifecycle management**: Load, execute, and close Lua scripts
- **Event callbacks**: Dispatch game events (frame updates, player actions, combat, environmental interactions) to Lua via `L_Call_*` functions
- **Entity lifecycle**: Notify Lua when monsters, projectiles, and objects are invalidated
- **Game state exposure**: Provide query functions for camera control, action queues, scoring modes, and player weapon state
- **Audio/rendering**: Manage Lua-related muting and texture palette access
- **Cutscene cameras**: Support timed camera paths with angle interpolation

## External Dependencies
- **cseries.h**: Core C++ utilities (types, macros, standard library wrappers)
- **world.h**: World coordinates (`world_point3d`, `world_distance`, angle types)
- **ActionQueues.h**: Player input queue management (`ActionQueues` class)
- **shape_descriptors.h**: Texture/sprite descriptor type (`shape_descriptor`)

# Source_Files/Lua/lua_templates.h
## File Purpose

Defines C++ template classes and functions for exposing game engine objects and containers to Lua scripts. This is the core Lua/C++ binding infrastructureΓÇöit wraps indexed game entities (monsters, items, players) and enumerations as Lua objects, allowing Lua scripts to query and manipulate them with automatic type checking and method dispatch.

## Core Responsibilities

- Provide template-based class registration (`L_Class`) to expose indexed objects to Lua
- Implement object wrapping and instance caching to maintain object identity across Lua calls
- Define validation and access predicates (`Valid`, `Length`) as pluggable callbacks
- Support enumeration types with mnemonic name-to-value mapping (`L_Enum`)
- Implement container-like access patterns (array indexing, iteration, method calls) via `L_Container`
- Manage Lua registry storage for method tables, instance caches, and mnemonic tables
- Provide metamethod handlers (`__index`, `__newindex`, `__tostring`, `__eq`) for Lua objects
- Support string-based (mnemonic) lookup for enum containers (`L_EnumContainer`)

## External Dependencies

- **Lua C API:** lua.h, lauxlib.h, lualib.h ΓÇö Core Lua embedding and auxiliary library functions (luaL_newmetatable, lua_pushcfunction, lua_pcall, etc.)
- **Boost:** boost/function.hpp ΓÇö Function wrapper for pluggable validation/length predicates
- **Game engine:** lua_script.h (game event callbacks), lua_mnemonics.h (lang_def mnemonic arrays), cseries.h (engine types and macros)
- **Standard library:** sstream (std::ostringstream for formatting __tostring output)

# Source_Files/Lua/luaconf.h
## File Purpose
Configuration header for the Lua interpreter that defines compile-time settings, platform-specific features, and behavioral parameters. Allows customizing Lua without modifying core source files via preprocessor macros marked with `@@` comments.

## Core Responsibilities
- Platform detection (Windows, Linux, macOS, POSIX compatibility)
- Define library search paths and module loading behavior
- Configure API visibility, linkage, and DLL export decorators
- Set memory management and garbage collection parameters
- Define numeric type representation and arithmetic operations
- Configure exception handling and error longjmp behavior
- Manage backward compatibility modes for older Lua versions
- Establish recursion depth, stack size, and variable limits
- Configure standard input handling and REPL prompts for standalone interpreter

## External Dependencies
**Conditional includes** (based on platform/build flags):
- `<limits.h>`, `<stddef.h>` ΓÇö always included
- `<unistd.h>` ΓÇö POSIX systems (isatty, mkstemp)
- `<io.h>`, `<stdio.h>` ΓÇö Windows
- `<assert.h>` ΓÇö when `LUA_USE_APICHECK` defined
- `<math.h>` ΓÇö when `LUA_CORE` defined
- `<readline/readline.h>`, `<readline/history.h>` ΓÇö when `LUA_USE_READLINE` defined

**Compiler/platform symbols used**:
- `__STRICT_ANSI__`, `__GNUC__`, `__ELF__`, `__cplusplus`, `_MSC_VER` ΓÇö compiler detection
- `__declspec(dllexport/dllimport)` ΓÇö MSVC DLL export
- `__attribute__((visibility("hidden")))` ΓÇö GCC/ELF visibility
- `INT_MAX`, `LONG_MAX`, `BUFSIZ` ΓÇö from standard headers

# Source_Files/Lua/lualib.h
## File Purpose
Header file declaring the Lua standard library initialization API. Defines constants for library names, declares functions to open individual standard libraries (base, table, io, os, string, math, debug, package), and provides a convenience function to open all libraries at once.

## Core Responsibilities
- Declare library opener functions (`luaopen_*`) for each standard library
- Define symbolic constants for library names (`LUA_*LIBNAME`)
- Define the file handle type key constant (`LUA_FILEHANDLE`)
- Declare the bulk library opener (`luaL_openlibs`)
- Provide assertion macro (`lua_assert`)

## External Dependencies
- **Includes:** `lua.h` (core Lua API, defines `lua_State`, constants, stack manipulation)
- **Macros used:** `LUALIB_API` (visibility/calling convention, defined in `luaconf.h`)
- **Types used (from lua.h):** `lua_State`

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

## External Dependencies
- **Includes**: `lobject.h` (provides `Proto`), `lzio.h` (provides `ZIO`, `Mbuffer`)
- **Types from elsewhere**: `lua_State` (lua.h), `lua_Writer` (lua.h)
- **Implementation files**: lundump.c (undump/header), ldump.c (dump), print.c (print)


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

## External Dependencies
- **`ldo.h`** ΓÇö Stack and call frame management (stack growth, call protocol).
- **`lobject.h`** ΓÇö Type definitions for Lua values, tagged unions, string/table/function structures.
- **`ltm.h`** ΓÇö Tag method (metamethod) dispatch and lookup.

# Source_Files/Lua/lzio.h
## File Purpose
Provides buffered input stream abstraction for Lua's I/O system. Enables efficient reading from various sources (files, strings, memory) through a pluggable reader callback and an internal buffer to minimize reader calls.

## Core Responsibilities
- Define buffered input stream structure (`Zio`) and callback-based reading protocol
- Provide dynamic memory buffer structure (`Mbuffer`) with resizing/allocation utilities
- Expose macros for efficient character-level reads with lazy filling
- Declare stream initialization and reading functions
- Separate public API from implementation details (marked as "Private Part")

## External Dependencies
- **lua.h**: `lua_State`, `lua_Reader` callback typedef (`const char *(*)(lua_State *L, void *ud, size_t *sz)`)
- **lmem.h**: `luaM_reallocvector` macro for memory management


