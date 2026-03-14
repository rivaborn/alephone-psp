# Source_Files/CSeries/csalerts.h

## File Purpose
Header file declaring alert/error handling and debug assertion facilities for the Aleph One game engine. Provides a cross-platform interface for displaying alerts, pausing execution, halting the program, and performing debug assertions with compiler-specific annotations (e.g., `noreturn` for GCC).

## Core Responsibilities
- Declare alert display functions with severity levels (info vs. fatal)
- Provide debug pause/halt entry points for interactive debugging
- Define assertion and warning macros with optional message variants
- Expose internal assertion implementation (`_alephone_assert`, `_alephone_warn`)
- Handle platform-specific branching (Mac Carbon, SDL, GCC vs. others)
- Supply compiler annotations (NORETURN) for static analysis

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| (anonymous enum) | enum | Severity levels: `infoError`, `fatalError` |

## Global / File-Static State
None.

## Key Functions / Methods

### alert_user (overload 1)
- Signature: `void alert_user(const char *message, short severity = infoError)`
- Purpose: Display an alert dialog with a message and severity level
- Inputs: C-string message, severity enum (defaults to infoError)
- Outputs/Return: void
- Side effects: User-visible dialog; blocks execution
- Calls: (implementation not visible)
- Notes: C++ default argument syntax in C header suggests C++ compilation mode

### alert_user (overload 2)
- Signature: `void alert_user(short severity, short resid, short item, OSErr error)`
- Purpose: Display an alert using resource IDs and an OS error code
- Inputs: severity level, resource ID, item ID, OSErr
- Outputs/Return: void
- Side effects: User-visible dialog; blocks execution
- Calls: (implementation not visible)
- Notes: Likely Mac-specific resource-based alert construction

### halt
- Signature: `void halt(void) NORETURN`
- Purpose: Terminate the program immediately
- Inputs: none
- Outputs/Return: none (noreturn)
- Side effects: Process termination
- Calls: (implementation not visible)
- Notes: Compiler hint prevents unreachable code warnings

### vhalt
- Signature: `void vhalt(const char *message) NORETURN`
- Purpose: Terminate the program after displaying a message
- Inputs: C-string message
- Outputs/Return: none (noreturn)
- Side effects: Displays message; terminates process
- Calls: (implementation not visible)

### _alephone_assert
- Signature: `void _alephone_assert(const char *file, long line, const char *what) NORETURN`
- Purpose: Handle assertion failure (called by assert macro)
- Inputs: source file, line number, assertion condition text
- Outputs/Return: none (noreturn)
- Side effects: Likely logs/displays error; terminates process
- Calls: (implementation not visible)
- Notes: Private implementation; wrapped by public macros

### _alephone_warn
- Signature: `void _alephone_warn(const char *file, long line, const char *what)`
- Purpose: Handle non-fatal warning (called by warn macro)
- Inputs: source file, line number, assertion condition text
- Outputs/Return: void
- Side effects: Likely logs/displays warning; execution continues
- Calls: (implementation not visible)

**Trivial helpers:** `pause_debug()`, `vpause()`, `SimpleAlert()` documented inline.

## Control Flow Notes
This file provides error/debugging infrastructure used throughout the engine:
- **Assert macros** (`assert`, `vassert`) are DEBUG-only; compiled to no-ops in Release builds
- **Warning macros** (`warn`, `vwarn`) similar but non-fatal
- **halt/vhalt** are unrecoverable shutdown paths
- **alert_user** functions support user-facing error messaging at runtime

## External Dependencies
- **System includes (not explicit):** OSErr (Mac/system error code)
- **Mac-specific types:** AlertType, DialogItemIndex (conditional on `TARGET_API_MAC_CARBON && !defined(SDL)`)
- **Compiler builtins:** `__attribute__((noreturn))` (GCC)
- **Preprocessor conditionals:** `__GNUC__`, `TARGET_API_MAC_CARBON`, `SDL`, `DEBUG`
