# Source_Files/Lua/ldebug.h

## File Purpose
Header file for Lua's debug interface module. Declares error reporting and code validation functions used throughout the interpreter, along with utility macros for managing program counters and debug hooks.

## Core Responsibilities
- Declare type error, concatenation error, arithmetic error, and runtime error reporting functions
- Provide program counter manipulation macros (`pcRel`)
- Provide line number lookup macro for debugging (`getline`)
- Declare code and instruction validation functions (`luaG_checkcode`, `luaG_checkopenop`)
- Declare debug hook management via `resethookcount` macro

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### luaG_typeerror
- Signature: `void luaG_typeerror(lua_State *L, const TValue *o, const char *opname)`
- Purpose: Report a type error when an operation is applied to an invalid type
- Inputs: Lua state `L`, offending value `o`, operation name string
- Outputs/Return: None (void; terminates via longjmp)
- Side effects: Raises error in VM via error handling mechanism
- Calls: Not inferable from header
- Notes: LUAI_FUNC indicates engine API boundary

### luaG_concaterror
- Signature: `void luaG_concaterror(lua_State *L, StkId p1, StkId p2)`
- Purpose: Report error when concatenation operator applied to invalid types
- Inputs: Lua state `L`, two stack indices with invalid values
- Outputs/Return: None (void; terminates via error mechanism)
- Side effects: Raises error
- Calls: Not inferable from header
- Notes: Specialized error for string concatenation

### luaG_aritherror
- Signature: `void luaG_aritherror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report error when arithmetic operation applied to non-numeric types
- Inputs: Lua state `L`, two invalid operand values
- Outputs/Return: None (void; terminates via error)
- Side effects: Raises error
- Calls: Not inferable from header

### luaG_ordererror
- Signature: `int luaG_ordererror(lua_State *L, const TValue *p1, const TValue *p2)`
- Purpose: Report error when comparison operator applied to incompatible types
- Inputs: Lua state `L`, two incomparable values
- Outputs/Return: Returns error code (int)
- Side effects: Raises error
- Calls: Not inferable from header
- Notes: Returns int unlike other error functions

### luaG_runerror
- Signature: `void luaG_runerror(lua_State *L, const char *fmt, ...)`
- Purpose: Report generic runtime error with formatted message
- Inputs: Lua state `L`, format string and variadic arguments
- Outputs/Return: None (void; terminates via error)
- Side effects: Raises error with formatted message
- Calls: Not inferable from header
- Notes: Variadic function for flexible error messages

### luaG_errormsg
- Signature: `void luaG_errormsg(lua_State *L)`
- Purpose: Process and dispatch error message from VM error handling
- Inputs: Lua state `L`
- Outputs/Return: None (void; terminates via error mechanism)
- Side effects: Processes error stack, invokes error handler
- Calls: Not inferable from header

### luaG_checkcode
- Signature: `int luaG_checkcode(const Proto *pt)`
- Purpose: Validate bytecode integrity of a function prototype
- Inputs: Function prototype pointer
- Outputs/Return: Returns validation result (int; likely 0=valid, 1=error)
- Side effects: None (const input, read-only)
- Calls: Not inferable from header
- Notes: Used in debugging/sanity checks

### luaG_checkopenop
- Signature: `int luaG_checkopenop(Instruction i)`
- Purpose: Validate that an instruction is a valid "open" operation (e.g., CALL, RETURN with variable results)
- Inputs: Single instruction value
- Outputs/Return: Returns validation result (int)
- Side effects: None
- Calls: Not inferable from header

## Control Flow Notes
These functions are called by the Lua interpreter during **execution/runtime** when errors occur or during debug validation phases. They are not part of init/frame/update loops; they are error paths and validation checkpoints invoked on demand by the VM core (`ldo.c`, `lvm.c`).

## External Dependencies
- **Includes**: `lstate.h` (Lua state structures, call stack info)
- **External symbols used**: `lua_State`, `TValue`, `StkId`, `Proto`, `Instruction` (all defined in included headers and `lobject.h`)
- **Macro uses**: `LUAI_FUNC` (visibility/calling convention marker for Lua API)
