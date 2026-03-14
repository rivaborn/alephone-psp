# Source_Files/CSeries/cstypes.h

## File Purpose
Portable type definitions and fixed-point arithmetic infrastructure for the Aleph One game engine. Abstracts platform-specific integer types (Mac/Carbon, BeOS, SDL) into consistent C types and provides fixed-point (16.16) math macros and utility constants.

## Core Responsibilities
- Define portable fixed-width integer types (`int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`) across Mac/BeOS/SDL platforms
- Declare `_fixed` type and fixed-point arithmetic macros (`FIXED_ONE`, `FIXED_INTEGERAL_PART`, etc.)
- Provide min/max constants for integer types (`INT16_MAX`, `INT32_MIN`, etc.)
- Define utility constants (`MEG`, `KILO`) and macro helpers (`FOUR_CHARS_TO_INT`)
- Handle platform-conditional includes and feature detection (OpenGL, config.h)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `_fixed` | typedef (int32) | Fixed-point arithmetic type (16.16 format) |
| `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32` | typedef | Platform-independent integer types |
| `TimeType` | typedef | Platform-dependent time representation (uint32 on Mac, time_t elsewhere) |
| `byte` | typedef (uint8) | Legacy byte alias |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NONE` | enum constant | global | Sentinel value (-1) |
| `UNONE` | enum constant | global | Unsigned sentinel (65535) |
| `MEG` | const int | global | 0x100000 (1 megabyte in bytes) |
| `KILO` | const int | global | 0x400L (1 kilobyte in bytes) |

## Key Functions / Methods
NoneΓÇöthis is a pure header file with typedefs and macros only.

## Control Flow Notes
Foundational infrastructure. Included by other headers to establish consistent integer and fixed-point types across all platform variations. No runtime control flow.

## External Dependencies
- **Conditional includes:**
  - `<Carbon/Carbon.h>` (Mac with EXPLICIT_CARBON_HEADER)
  - `<support/SupportDefs.h>` (BeOS)
  - `<SDL_types.h>`, `<time.h>` (SDL)
- **Always included:** `<limits.h>`, `config.h` (if HAVE_CONFIG_H)
- **Fallback:** Defines `GLfloat` as `float` if `HAVE_OPENGL` is not defined

## Notes
- Uses enum instead of const int for `NONE`/`UNONE` to save TOC (Table of Contents) space on embedded systems (IR note in header).
- Fixed-point format is 16.16 (16 fractional bits); shift macros assume this layout.
- `_fixed` name chosen to avoid MSVC namespace conflicts.
- Supports legacy `byte` typedef but this is marked for potential removal ("Hmmm, this should be removed one day...").
