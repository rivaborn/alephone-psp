# Source_Files/Misc/psp_common.h

## File Purpose
Header file providing input utility macros for PSP (PlayStation Portable) gamepad handling. Defines a single macro for checking if a button on a gamepad controller is currently pressed.

## Core Responsibilities
- Define input state checking macro for PSP gamepad buttons
- Provide simple bitwise abstraction for button detection

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### PSP_PRESSED
- Signature: `#define PSP_PRESSED(pad, button) (pad.Buttons & button)`
- Purpose: Check if a specific button is currently pressed on a gamepad.
- Inputs:
  - `pad`: Gamepad state structure (type not defined in this file; assumed struct with `Buttons` field)
  - `button`: Button constant to test (typically a bit flag)
- Outputs/Return: Non-zero if button is pressed, zero otherwise (boolean-like).
- Side effects: None; read-only operation.
- Calls: None.
- Notes: Uses bitwise AND (`&`) to test if a button's bit is set. Treats any non-zero result as true; common idiom in C for flag testing. The `pad` structure and button constants are defined elsewhere.

## Control Flow Notes
Used during input polling or event handling in frame/update loops to query gamepad state. No control flow is visible in this file.

## External Dependencies
- `pad` structure type: defined elsewhere (not inferable from this file)
- Button constants: defined elsewhere
- Standard C header guard (`#ifndef`, `#define`, `#endif`)
