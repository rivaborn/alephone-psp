# Source_Files/Misc/Logging_gruntwork.h

## File Purpose
Auto-generated header providing convenience macros for logging at eight severity levels (Fatal, Error, Warning, Anomaly, Note, Summary, Trace, Dump). Supports both main-thread and non-main-thread variants, with variadic and fixed-argument forms.

## Core Responsibilities
- Define convenience macros delegating to `GetCurrentLogger()->logMessage()` and `GetCurrentLogger()->logMessageNMT()`
- Support variadic logging via `__VA_ARGS__` for flexible argument counts
- Support fixed 1ΓÇô5 argument logging through numbered macro variants (e.g., `logError1()`, `logError5()`)
- Provide non-main-thread (NMT) variants for thread-safe logging
- Provide context-scoped logging macros (`logContext*`) that instantiate temporary `LogContext` objects
- Auto-inject file, line, domain, and severity into all log calls

## Key Types / Data Structures
None defined here; types used:
| Name | Kind | Purpose |
|------|------|---------|
| `LogContext` | class/struct (external) | RAII scoped logging context |

## Global / File-Static State
None ΓÇö this file only defines macros; no static data.

## Key Functions / Methods
All macros delegate to two logger methods (defined elsewhere):

### `GetCurrentLogger()->logMessage(...)`
- Purpose: Main-thread logging
- Signature (conceptual): `logMessage(domain, level, file, line, message, [args...])`
- Called by: All unqualified `logFatal`, `logError`, etc. macros and numbered variants

### `GetCurrentLogger()->logMessageNMT(...)`
- Purpose: Non-main-thread safe logging
- Signature (conceptual): `logMessageNMT(domain, level, file, line, message, [args...])`
- Called by: All `logFatalNMT`, `logErrorNMT`, etc. macros and numbered variants

### `LogContext` constructor
- Purpose: Construct scoped logging context (RAII pattern)
- Called by: `logContext*` and `logContextNMT*` macros
- Parameters: `(is_nmt_flag, file, line, message, [args...])`
- Notes: Instantiated with unique name via `makeUniqueIdentifier(_theLogContext, __LINE__)` to allow nested contexts in the same scope

## Control Flow Notes
Not inferable ΓÇö this is a macro definition header with no runtime logic.

## External Dependencies
- **Function/macro**: `GetCurrentLogger()` ΓÇö returns logger singleton
- **Type**: `LogContext` ΓÇö scoped context object (defined elsewhere)
- **Constants**: `logDomain`, `logFatalLevel`, `logErrorLevel`, `logWarningLevel`, `logAnomalyLevel`, `logNoteLevel`, `logSummaryLevel`, `logTraceLevel`, `logDumpLevel` ΓÇö all defined elsewhere
- **Macro**: `makeUniqueIdentifier()` ΓÇö generates unique symbol names (defined elsewhere)
- **Standard macros**: `__FILE__`, `__LINE__`, `__VA_ARGS__`

**Notes:**
- File is auto-generated; edit `aleph/tools/gen-Logging_gruntwork.csh` instead.
- Inactive `#if 0` branch shows obsolete single-message design; active `#else` uses variadic macros.
- Numbered variants (1ΓÇô5 args) support pre-variadic code or explicit arity control.
- NMT variants assume caller may be off main thread and dispatch to thread-safe logger method.
