# Source_Files/Misc/psp_logging.h

## File Purpose
Defines platform-specific logging macros for PlayStation Portable (PSP) builds. Provides conditional debug output that stringifies file paths and line numbers at compile time, with no-op fallback for non-PSP platforms.

## Core Responsibilities
- Define string stringification macros for token-to-string conversion
- Provide a conditional logging macro (`PSP_STRLOG`) that outputs to stdout on PSP
- Support compile-time file/line information capture via preprocessor
- Allow clean disable of logging for non-PSP builds

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### STRINGIFY
- Macro that converts a preprocessor token into a string literal using the `#` operator.
- Purpose: Convert arbitrary tokens to their string representation.
- Inputs: `x` ΓÇô any preprocessor token
- Outputs/Return: String literal representation of `x`

### STRINGIFY_MACRO
- Macro that wraps `STRINGIFY` to force macro expansion before stringification.
- Purpose: Handle stringification of macro-expanded values (indirect stringification).
- Inputs: `x` ΓÇô token or macro
- Outputs/Return: String literal after macro expansion

### PSP_STRLOG
- Conditional logging macro; expands to `printf()` call on PSP, no-op elsewhere.
- Purpose: Output diagnostic messages with file and line context only on PSP platform.
- Inputs: `str` ΓÇô string message to log
- Outputs/Return: None (void macro)
- Side effects: Calls `printf()` to stdout on PSP (formatted as `"PSP: <str> - <FILE>:<LINE>"`); no output on non-PSP
- Calls: `printf()` (when PSP is defined)
- Notes: Empty definition when `PSP` macro is undefined; allows code to use `PSP_STRLOG()` unconditionally

## Control Flow Notes
This file operates purely at compile time via preprocessor directives. When `PSP` is defined (PSP build), `PSP_STRLOG()` becomes active logging; otherwise it's stripped to nothing. No runtime control flow.

## External Dependencies
- Standard C library: `printf()` (implicit via `stdio.h`, included elsewhere when PSP is active)
- Preprocessor: `#define`, conditional compilation (`#ifdef`)
