# Source_Files/LibNAT/utility.h

## File Purpose
Defines common utility constants and function declarations for the LibNAT library. Provides standardized size limits for HTTP/URL handling, port numbers, and C-string operations used throughout the project.

## Core Responsibilities
- Define standard C-string handling constants (null terminators)
- Specify constraints for port numbers (max value and string length)
- Define HTTP protocol constants (status codes, protocol prefix, default port)
- Define size limits for URL components (URLs, hostnames, resource paths)
- Declare string utility functions (case conversion)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Str_To_Upper
- **Signature:** `int LNat_Str_To_Upper(const char * str, char * dest)`
- **Purpose:** Convert a null-terminated C-string to uppercase.
- **Inputs:** 
  - `str`: source C-string (const, null-terminated)
  - `dest`: destination buffer for uppercase result
- **Outputs/Return:** Integer status code (OK on success; specific error code on failure not detailed in header)
- **Side effects:** Writes to destination buffer; no global state modification visible.
- **Calls:** Not inferable from this file (implementation in separate .c file).
- **Notes:** Assumes `dest` is pre-allocated and large enough; no buffer overflow protection evident in signature.

## Control Flow Notes
This is a utility/infrastructure header included by multiple subsystems. Not part of main game loop (init/frame/render/shutdown). Likely included early in compilation to provide common constants and helper declarations.

## External Dependencies
- Standard C conventions only.
- No `#include` directives shown; expected to be included by other translation units that need these constants and the string utility function.
- Function implementation defined elsewhere (presumably `utility.c`).
