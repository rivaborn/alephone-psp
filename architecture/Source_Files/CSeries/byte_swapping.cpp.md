# Source_Files/CSeries/byte_swapping.cpp

## File Purpose
Implements byte-swapping routines for endianness conversion on little-endian systems. Provides the core logic to reverse byte order in 2-byte and 4-byte fields during game data I/O operations.

## Core Responsibilities
- Swap bytes in 2-byte (16-bit) values in memory buffers
- Swap bytes in 4-byte (32-bit) values in memory buffers
- Process multiple sequential fields of the same type
- Conditionally compile only on little-endian architectures

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_bs_field` | typedef (short) | Enum-like type indicating field size for swapping |

## Global / File-Static State
None.

## Key Functions / Methods

### byte_swap_memory
- **Signature:** `void byte_swap_memory(void *memory, _bs_field type, int fieldcount)`
- **Purpose:** Reverses byte order in a memory buffer to convert between little-endian and big-endian representations.
- **Inputs:**
  - `memory`: pointer to buffer containing fields to swap
  - `type`: field type (`_2byte` = -2 or `_4byte` = -4)
  - `fieldcount`: number of fields to swap (may become negative per code comment, but loop checks `> 0`)
- **Outputs/Return:** void (modifies memory in-place)
- **Side effects:** Mutates the buffer at `memory`
- **Calls:** None
- **Notes:**
  - Entire function wrapped in `#ifdef ALEPHONE_LITTLE_ENDIAN`; on big-endian systems, call is macro'd to no-op
  - Implements byte-by-byte swaps via `uint8*` pointer arithmetic
  - For `_2byte`: swaps positions [0] Γåö [1]
  - For `_4byte`: swaps [0] Γåö [3] and [1] Γåö [2] (full reverse)
  - No validation for null pointers or invalid type values

## Control Flow Notes
This is a utility function invoked during game file I/O (load/save of binary map/level data). The conditional compilation ensures it only compiles on little-endian targets; big-endian systems use the macro stub in the header.

## External Dependencies
- `#include "cseries.h"` ΓÇô provides `uint8` type and endianness detection (`ALEPHONE_LITTLE_ENDIAN`)
- `#include "byte_swapping.h"` ΓÇô declares function signature and `_bs_field` enum
