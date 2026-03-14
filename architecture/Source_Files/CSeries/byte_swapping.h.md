# Source_Files/CSeries/byte_swapping.h

## File Purpose
Header for endianness-aware byte swapping operations in Aleph One. Provides a conditional interface to swap byte order in memory blocks, with platform-specific behavior controlled by compile-time flags.

## Core Responsibilities
- Define field type marker for byte swap operations (`_bs_field`)
- Declare the `byte_swap_memory()` function for swapping multi-byte values
- Provide conditional macro wrapping that optimizes away on little-endian platforms
- Support cross-platform data format compatibility (big/little endian conversion)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_bs_field` | typedef | Short integer used as tag/marker for swap field type specification |
| (enum) `_2byte`, `_4byte` | enum constants | Specify 2-byte and 4-byte field sizes for swapping |

## Global / File-Static State
None.

## Key Functions / Methods

### byte_swap_memory
- **Signature:** `void byte_swap_memory(void *memory, _bs_field type, int fieldcount)`
- **Purpose:** Swap byte order for a block of same-typed fields in memory (e.g., convert big-endian shorts to little-endian).
- **Inputs:**
  - `memory`: pointer to buffer containing fields to swap
  - `type`: field size marker (`_2byte` or `_4byte`)
  - `fieldcount`: number of fields to swap
- **Outputs/Return:** None (void, modifies in-place)
- **Side effects:** Modifies memory in-place; I/O none.
- **Calls:** (definition elsewhere)
- **Notes:** Function is conditionally compiled out (replaced with no-op macro) if `ALEPHONE_LITTLE_ENDIAN` is defined, since little-endian systems don't need swapping.

## Control Flow Notes
This is a utility header with conditional compilation logic:
- **If `ALEPHONE_LITTLE_ENDIAN` is defined:** the macro becomes a no-op `((void)0)`, so byte swapping is skipped entirely.
- **If not defined:** the actual function is called, implying a big-endian platform that needs conversion.
Used during init/load operations when reading data files that may be in non-native endianness.

## External Dependencies
- `<stddef.h>` ΓÇö standard library (likely for size/offset definitions, though not directly used in this file)
- `byte_swap_memory()` implementation ΓÇö defined elsewhere (likely in a .c file in CSeries/)
