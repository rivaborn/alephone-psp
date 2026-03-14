# Source_Files/CSeries/cseries.h

## File Purpose
Master include header for the CSeries cross-platform compatibility library. Aggregates type definitions, macros, and utility headers while providing platform-specific shims for macOS APIs, endianness handling, and compiler-specific namespace workarounds.

## Core Responsibilities
- Central aggregation point for all CSeries subsystem headers
- Platform detection and compiler-specific configuration (MSVC, GCC, Clang)
- Endianness abstraction via SDL byte-order detection
- macOS legacy API compatibility layer (Rect, OSErr, Str255, RGBColor)
- Conditional inclusion of macOS frameworks (CoreFoundation, Carbon, Quickdraw)
- Namespace management for C++ compilers with different std:: handling
- Build configuration injection (VERSION, feature flags from config.h)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `RGBColor` | struct | RGB color representation (16-bit per channel) |
| `Rect` | struct | Rectangle with top, left, bottom, right coordinates (macOS compat) |
| `OSErr` | typedef | OS error code type (platform-specific) |
| `Str255` | typedef | 256-byte Pascal-style string (platform-specific) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ALEPHONE_LITTLE_ENDIAN` | #define | conditional global | Set to 1 if platform uses little-endian byte order |
| `kFontIDMonaco` | const int | conditional global | Font ID constant for Monaco font (value 4) |
| `kFontIDCourier` | const int | conditional global | Font ID constant for Courier font (value 22) |
| `VERSION` | #define | global | Build version string (from config.h or "unknown version") |
| `DEBUG` | #define | global | Debug mode flag (unconditionally defined) |

## Key Functions / Methods
None. This is a pure header file with no function implementations.

## Control Flow Notes
This file establishes the compile-time foundation for the Aleph One engine. It runs once at the start of compilation, detecting platform capabilities, setting up type aliases, and including all CSeries subsystem headers. The conditional includes based on macOS detection and the SDL_RFORK_HACK flag suggest support for both classic and modern macOS toolchains. Compiler-specific workarounds (e.g., MSVC for-loop redeclaration fix, namespace handling) are applied at this point.

## External Dependencies

**Direct includes:**
- `<SDL.h>`, `<SDL_byteorder.h>` ΓÇö SDL cross-platform library
- `<time.h>` ΓÇö Standard time functions
- `<string>` ΓÇö C++ std::string
- `config.h` ΓÇö Auto-generated build configuration (feature flags, version)

**Conditional includes:**
- `<CoreFoundation/CoreFoundation.h>` ΓÇö macOS CoreFoundation APIs (on __APPLE__ && __MACH__)
- `<MacTypes.h>`, `<Quickdraw.h>` ΓÇö Classic macOS toolbox APIs (when SDL_RFORK_HACK defined)

**CSeries subsystem headers (included at end):**
- `cstypes.h` ΓÇö Fixed-size integer types, fixed-point math, platform type aliases
- `csmacros.h` ΓÇö MIN/MAX, bit manipulation, bounds-checking templates
- `cscluts.h` ΓÇö Color table structures and functions
- `csstrings.h` ΓÇö String manipulation (Pascal/C-string conversions, formatting)
- `csfonts.h` ΓÇö Font specifications and styling constants
- `cspixels.h` ΓÇö Pixel format conversions (8/16/32-bit color)
- `csalerts.h` ΓÇö Alert/dialog types (not shown)
- `csdialogs.h` ΓÇö Dialog handling (not shown)
- `csmisc.h` ΓÇö Miscellaneous utilities (not shown)
