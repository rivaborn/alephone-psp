# Source_Files/Misc/game_errors.cpp

## File Purpose
Implements a simple global error tracking system that maintains the last error type and code. Provides functions to set, query, check, and clear the current error state. This is a utility module for propagating error information across the game engine.

## Core Responsibilities
- Maintain static storage for the most recent error type and error code
- Provide getter/setter functions for error state
- Validate error types and codes in debug builds
- Allow queries to check if an error is pending
- Support clearing the error state

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|------|---------|
| `last_type` | `short` | static | Stores the type of the last error (systemError or gameError) |
| `last_error` | `short` | static | Stores the error code of the last error |

## Key Functions / Methods

### set_game_error
- Signature: `void set_game_error(short type, short error_code)`
- Purpose: Set the current error state with a type and error code.
- Inputs: `type` (error type: systemError or gameError), `error_code` (specific error code)
- Outputs/Return: None (void)
- Side effects: Updates global static variables `last_type` and `last_error`
- Calls: `assert()` (from cseries.h)
- Notes: Asserts that type is valid (0 to NUMBER_OF_TYPES-1). In DEBUG builds, additionally asserts that gameError codes are valid (0 to NUMBER_OF_GAME_ERRORS-1).

### get_game_error
- Signature: `short get_game_error(short *type)`
- Purpose: Retrieve the current error code and optionally the error type.
- Inputs: `type` (nullable pointer to short where type will be written)
- Outputs/Return: Returns the last error code; if type pointer is non-null, dereferences it and stores the last error type
- Side effects: None (read-only)
- Calls: None
- Notes: Caller may pass NULL for type if only the error code is needed.

### error_pending
- Signature: `bool error_pending(void)`
- Purpose: Check if an error is currently set.
- Inputs: None
- Outputs/Return: Returns true if `last_error` is non-zero (indicating a pending error)
- Side effects: None (read-only)
- Calls: None
- Notes: Uses simple non-zero check; does not consider error type.

### clear_game_error
- Signature: `void clear_game_error(void)`
- Purpose: Clear the current error state.
- Inputs: None
- Outputs/Return: None (void)
- Side effects: Resets both `last_error` and `last_type` to 0
- Calls: None
- Notes: Sets both fields to 0; type is reset to 0 (not explicitly documented, but occurs in code).

## Control Flow Notes
This module is not part of the frame loop or initialization sequence. It is a utility called opportunistically by other engine systems to record and communicate error state. No entry point visible in this file; error tracking is typically accessed through getter/setter functions from elsewhere in the codebase.

## External Dependencies
- `cseries.h` ΓÇö provides platform abstractions, assert macro, and type definitions (`short`, `bool`)
- `game_errors.h` ΓÇö header defining the public API and error type/code enums
