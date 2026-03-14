# Source_Files/CSeries/csstrings.h

## File Purpose
Header declaring string manipulation and encoding utilities for the Aleph One game engine. Provides Pascal string (pstring) and C string operations, printf-style formatting functions, resource-based string loading, and Mac Roman Γåö Unicode/UTF-8 character encoding conversions.

## Core Responsibilities
- Load strings from resource sets by ID (Pascal and C string variants)
- Manage global temporary string buffer (`temporary[256]`)
- Convert between Pascal strings, C strings, and `std::string`
- Provide printf-style formatting (debug output, file logging)
- Transform character encodings (Mac Roman Γåö Unicode, Mac Roman Γåö UTF-8)
- Support string duplication and bounded copying
- Build string vectors from resource sets

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Str255` | typedef | Pascal string type (256-byte length-prefixed string) |
| `temporary` | char[256] | Global temporary buffer for string operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `temporary` | `char[256]` | global | Reusable scratch buffer for temporary string work |
| `ptemporary` | macro ΓåÆ `Str255*` | global | Cast overlay of `temporary` as Pascal string |

## Key Functions / Methods

### countstr
- Signature: `extern size_t countstr(short resid)`
- Purpose: Count the number of strings in a resource set
- Inputs: `resid` ΓÇô resource ID
- Outputs: Number of strings in set
- Side effects: None
- Calls: Likely resource manager (not visible)

### getpstr / getcstr
- Signature: `extern unsigned char *getpstr(unsigned char *string, short resid, size_t item)` and C string variant
- Purpose: Retrieve a specific string from a resource set into a buffer
- Inputs: `string` (destination buffer), `resid` (resource ID), `item` (string index)
- Outputs: Pointer to destination buffer
- Side effects: Writes to destination buffer
- Calls: Resource manager (not visible)

### pstrcpy / pstrncpy / pstrdup
- Purpose: Copy Pascal strings with optional length limit or allocate duplicate
- Inputs: Source/destination Pascal strings, optional byte count
- Outputs: Pointer to destination (or allocated buffer for `pstrdup`)
- Side effects: Memory allocation for `pstrdup`
- Notes: `pstrcpy` marked const-correct on source; `pstrdup` allocates

### a1_c2pstr / a1_p2cstr
- Purpose: In-place conversion between C and Pascal string formats
- Inputs: Buffer containing one format
- Outputs: Same buffer in opposite format
- Side effects: Modifies input buffer in-place
- Notes: Dangerous if buffer sizes don't accommodate length prefix

### csprintf / psprintf
- Purpose: Printf-style formatting into C or Pascal string buffers
- Inputs: Destination buffer, format string, variadic args
- Outputs: Pointer to destination buffer
- Side effects: Writes formatted string; uses PRINTF_STYLE_ARGS for compiler warnings
- Notes: Compiler validates format string at compile-time (GCC/Clang)

### dprintf / fdprintf
- Purpose: Debug logging to console (dprintf) or file (fdprintf, writes to AlephOneDebugLog.txt)
- Inputs: Format string, variadic args
- Outputs: None
- Side effects: Console output or file I/O; marked with PRINTF_STYLE_ARGS

### String/Encoding Conversions
- `copy_string_to_pstring`, `copy_string_to_cstring` ΓÇô convert `std::string` ΓåÆ Pascal/C strings with max length
- `pstring_to_string` ΓÇô convert Pascal string ΓåÆ `std::string`
- `mac_roman_to_unicode` (4 overloads) ΓÇô character or string encoding (single char, string with/without limit)
- `unicode_to_mac_roman` ΓÇô reverse conversion
- `mac_roman_to_utf8`, `utf8_to_mac_roman` ΓÇô full encoding transforms

## Control Flow Notes
This is a utility header with no inherent flow. Functions are called:
- **Initialization**: Resource string loading via `getpstr`/`getcstr`
- **Runtime**: Printf-style functions for debug output and logging
- **Rendering/UI**: String conversions before text display
- **Data I/O**: Encoding conversions during serialization/file load

## External Dependencies
- **Includes**: `cstypes.h` (typed integers), `<string>`, `<vector>` (STL)
- **Compiler attribute**: `PRINTF_STYLE_ARGS` macro (GCC format attribute for type checking)
- **Defined elsewhere**: Resource manager (provides `resid` lookup), Mac Roman encoding tables (for char conversion)
- **Notes**: Implicit dependency on a resource system (Classic Mac-like string resources by ID)
