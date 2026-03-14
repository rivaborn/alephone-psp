# Source_Files/CSeries/snprintf.cpp

## File Purpose
Provides portable fallback implementations of `snprintf()` and `vsnprintf()` for platforms that lack them. Acts as a compatibility layer that wraps `vsprintf()` and detects buffer overflows with logging warnings.

## Core Responsibilities
- Provide standards-compliant `snprintf()` wrapper via variadic argument delegation
- Implement `vsnprintf()` with post-hoc overflow detection and logging
- Conditionally compile only on platforms without native implementations
- Prevent recursive warning logs during overflow detection

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `issuingWarning` | `static bool` | file-static | Re-entrancy guard to prevent recursive logging when vsnprintf itself calls log functions |

## Key Functions / Methods

### snprintf
- **Signature:** `int snprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, ...)`
- **Purpose:** Variadic wrapper around vsnprintf; delegates to core implementation.
- **Inputs:** target buffer, buffer size, format string, variadic arguments
- **Outputs/Return:** character count written (from vsnprintf)
- **Side effects:** none directly; side effects occur in vsnprintf
- **Calls:** `va_start()`, `vsnprintf()`, `va_end()`
- **Notes:** Conditional compilation (`#ifndef HAVE_SNPRINTF`); simple forwarding function

### vsnprintf
- **Signature:** `int vsnprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, va_list inArgs)`
- **Purpose:** Core implementation; formats string via vsprintf and detects buffer overruns post-facto.
- **Inputs:** target buffer, buffer size, format string, va_list of arguments
- **Outputs/Return:** character count written (from vsprintf, not clamped to buffer size)
- **Side effects:** Calls `vsprintf()` (unsafeΓÇöwrites before bounds check); may call `logWarning2()` if overflow detected; modifies `issuingWarning` static flag
- **Calls:** `vsprintf()`, `logWarning2()`
- **Notes:** Conditional compilation (`#ifndef HAVE_VSNPRINTF`); overflow detection is post-hoc and does not prevent the overflow itself; `issuingWarning` flag prevents re-entrance during logging

## Control Flow Notes
Pure utility layer with no initialization/shutdown. Call chain: `snprintf()` ΓåÆ `vsnprintf()` ΓåÆ `vsprintf()`. Overflow detection is logged but does not prevent buffer corruptionΓÇö`vsprintf()` has already written by the time the check occurs.

## External Dependencies
- **Includes:** `<stdio.h>` (vsprintf assumed available), `Logging.h` (logWarning2)
- **Defined elsewhere:** `vsprintf()` (C standard library), `logWarning2()` (Logging subsystem)
- **Conditional directives:** `HAVE_SNPRINTF`, `HAVE_VSNPRINTF` control compilation
