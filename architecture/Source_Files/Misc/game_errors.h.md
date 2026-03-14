# Source_Files/Misc/game_errors.h

## File Purpose
Header file defining error type categories and specific game error codes. Provides a simple error management API for setting, querying, and clearing errors throughout the game engine.

## Core Responsibilities
- Define error type enumeration (system vs. game errors)
- Define specific game error codes (file not found, out of range, sync failures, etc.)
- Declare error state management functions (set, get, clear)
- Provide error pending query for polling

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (unnamed) | enum | Error types: `systemError`, `gameError` |
| (unnamed) | enum | Game-specific error codes: `errNone`, `errMapFileNotSet`, `errIndexOutOfRange`, etc. |

## Global / File-Static State
None (state is implied to exist elsewhere, likely in a corresponding `.c` implementation file).

## Key Functions / Methods

### set_game_error
- Signature: `void set_game_error(short type, short error_code)`
- Purpose: Record an error of a specific type and code
- Inputs: error type (system or game), error code identifier
- Outputs/Return: void
- Side effects: Modifies global error state
- Calls: Not inferable from this file
- Notes: Single active error modelΓÇöoverwrites previous error

### get_game_error
- Signature: `short get_game_error(short *type)`
- Purpose: Retrieve the current error code and populate its type
- Inputs: pointer to type variable (output parameter)
- Outputs/Return: error code; type written to pointer
- Side effects: None (read-only query)
- Calls: Not inferable from this file
- Notes: Caller must provide valid pointer

### error_pending
- Signature: `bool error_pending(void)`
- Purpose: Check if an error is currently recorded
- Inputs: none
- Outputs/Return: true if error exists, false otherwise
- Side effects: None
- Calls: Not inferable from this file

### clear_game_error
- Signature: `void clear_game_error(void)`
- Purpose: Reset error state to none
- Inputs: none
- Outputs/Return: void
- Side effects: Clears global error state
- Calls: Not inferable from this file

## Control Flow Notes
Module functions as a global error query/reporting system. Likely called on init to clear state, during game loops to check for errors, and in error handlers to set/retrieve error details. The single-error model suggests errors are processed serially or errors take priority (later errors overwrite).

## External Dependencies
- No includes or external dependencies
