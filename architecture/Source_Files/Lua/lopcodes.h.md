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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OpMode` | enum | Instruction format: iABC (A+B+C), iABx (A+Bx), iAsBx (A+signed Bx) |
| `OpCode` | enum | All 38 VM opcodes: OP_MOVE, OP_LOADK, OP_CALL, OP_RETURN, OP_FORLOOP, OP_CLOSURE, etc. |
| `OpArgMask` | enum | Argument mode: N (unused), U (used), R (register/jump), K (constant/reg) |
| `Instruction` | typedef | Unsigned 32-bit word (from llimits.h) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `luaP_opmodes` | const lu_byte[] | extern | Per-opcode property mask (mode, B/C arg modes, sets A, is test) |
| `luaP_opnames` | const char*[] | extern | String names of all opcodes (for debugging/disassembly) |

## Key Functions / Methods
None. This file contains only type definitions, macros, and extern declarations.

## Macro Summary
**Field extraction**: GET_OPCODE, GETARG_A/B/C/Bx/sBx (shift and mask instruction words)  
**Field setting**: SET_OPCODE, SETARG_A/B/C/Bx/sBx (reconstruct instruction with new field)  
**Instruction creation**: CREATE_ABC, CREATE_ABx (assemble all fields into instruction)  
**Utilities**: MASK1/MASK0 (bit masks), BITRK/ISK/INDEXK/RKASK (register/constant encoding)  
**Properties**: getOpMode, getBMode, getCMode, testAMode, testTMode (query opcode metadata)

## Control Flow Notes
Static definition file loaded during compilation. Not part of init/frame/render loops; instead defines the bytecode instruction set that the VM's main loop (in another file) will fetch-decode-execute.

## External Dependencies
- `#include "llimits.h"` ΓÇö defines `Instruction` type (lu_int32), `lu_byte`, `cast()` macro, `MAX_INT`, `LUAI_BITSINT`, and `LUAI_DATA` annotation
- References to `lua.h` in copyright notice (not included here)
- External data `luaP_opmodes[NUM_OPCODES]` and `luaP_opnames[NUM_OPCODES+1]` ΓÇö defined elsewhere (likely lopcodes.c)
