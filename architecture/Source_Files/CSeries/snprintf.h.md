# Source_Files/CSeries/snprintf.h

## File Purpose
Compatibility shim header that conditionally declares `snprintf()` and `vsnprintf()` functions. Provides fallback function signatures for platforms lacking C99 standard library implementations, with build-time detection via `config.h`.

## Core Responsibilities
- Conditionally declare `snprintf()` for platforms without native support
- Conditionally declare `vsnprintf()` for platforms without native support
- Gate declarations on `HAVE_SNPRINTF` and `HAVE_VSNPRINTF` autoconf flags
- Provide portable printf-family formatting interface across build targets
- Include MSVC7 workaround (`<streambuf>`)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### snprintf
- Signature: `int snprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, ...)`
- Purpose: Format variadic arguments into a bounded buffer (printf-like behavior)
- Inputs: destination buffer, buffer size, format string, variadic arguments
- Outputs/Return: number of characters written (or would be written)
- Side effects: Writes formatted string to `inBuffer` up to `inBufferSize` bytes
- Calls: Not inferable (declaration only)
- Notes: Declared only if `HAVE_SNPRINTF` is undefined; otherwise omitted to use system implementation

### vsnprintf
- Signature: `int vsnprintf(char* inBuffer, size_t inBufferSize, const char* inFormat, va_list inArgs)`
- Purpose: Format va_list arguments into a bounded buffer (variadic-list variant)
- Inputs: destination buffer, buffer size, format string, va_list arguments
- Outputs/Return: number of characters written (or would be written)
- Side effects: Writes formatted string to `inBuffer` up to `inBufferSize` bytes
- Calls: Not inferable (declaration only)
- Notes: Declared only if `HAVE_VSNPRINTF` is undefined; otherwise omitted to use system implementation

## Control Flow Notes
Pure header file; no executable control flow. Preprocessor conditionals (`#ifndef` guards) determine whether declarations are included based on autoconf-generated `config.h` flags. Used during compilation to inject fallback signatures when the target platform lacks native snprintf/vsnprintf.

## External Dependencies
- `<stdarg.h>` ΓÇö va_list type (C standard library)
- `<streambuf>` ΓÇö MSVC7 strangeness workaround (C++ header, unusual in C codebase)
- `"config.h"` ΓÇö autoconf build-time feature detection (defines `HAVE_SNPRINTF`, `HAVE_VSNPRINTF`)

---

**Notes on context (from bundled `config.h`):**  
This is part of **Aleph One** (Marathon-engine game port to SDL, v20080721). Build configuration shows `HAVE_SNPRINTF` and `HAVE_VSNPRINTF` both defined (target has native support), so both conditional declarations are **skipped**. Header is defensive for older/non-POSIX platforms.
