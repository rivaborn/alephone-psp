# Source_Files/CSeries/byte_swapping.cpp
## File Purpose
Implements byte-swapping routines for endianness conversion on little-endian systems. Provides the core logic to reverse byte order in 2-byte and 4-byte fields during game data I/O operations.

## Core Responsibilities
- Swap bytes in 2-byte (16-bit) values in memory buffers
- Swap bytes in 4-byte (32-bit) values in memory buffers
- Process multiple sequential fields of the same type
- Conditionally compile only on little-endian architectures

## External Dependencies
- `#include "cseries.h"` ΓÇô provides `uint8` type and endianness detection (`ALEPHONE_LITTLE_ENDIAN`)
- `#include "byte_swapping.h"` ΓÇô declares function signature and `_bs_field` enum

# Source_Files/CSeries/byte_swapping.h
## File Purpose
Header for endianness-aware byte swapping operations in Aleph One. Provides a conditional interface to swap byte order in memory blocks, with platform-specific behavior controlled by compile-time flags.

## Core Responsibilities
- Define field type marker for byte swap operations (`_bs_field`)
- Declare the `byte_swap_memory()` function for swapping multi-byte values
- Provide conditional macro wrapping that optimizes away on little-endian platforms
- Support cross-platform data format compatibility (big/little endian conversion)

## External Dependencies
- `<stddef.h>` ΓÇö standard library (likely for size/offset definitions, though not directly used in this file)
- `byte_swap_memory()` implementation ΓÇö defined elsewhere (likely in a .c file in CSeries/)

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

## External Dependencies
- **System includes (not explicit):** OSErr (Mac/system error code)
- **Mac-specific types:** AlertType, DialogItemIndex (conditional on `TARGET_API_MAC_CARBON && !defined(SDL)`)
- **Compiler builtins:** `__attribute__((noreturn))` (GCC)
- **Preprocessor conditionals:** `__GNUC__`, `TARGET_API_MAC_CARBON`, `SDL`, `DEBUG`

# Source_Files/CSeries/csalerts_sdl.cpp
## File Purpose
Implements alert, warning, and assertion messaging for the Aleph One engine with SDL-based UI. Provides user-facing error dialogs with word-wrapping, stderr fallback for headless execution, and integration with the engine's logging system. Handles fatal errors and debugging breakpoints.

## Core Responsibilities
- Display modal alert dialogs with multi-line text wrapping to `MAX_ALERT_WIDTH`
- Fallback stderr output when no SDL video surface is available
- Log all alerts/warnings/errors to the logging system with severity levels
- Handle fatal errors by recording state, then aborting or dereferencing null pointer
- Support assertion failures with file/line information
- Provide pause points for debugging without exiting (pause_debug, vpause)
- Gate game window updates on alert severity (info vs. fatal)

## External Dependencies
- **Headers:** `<stdio.h>`, `"cseries.h"`, `"Logging.h"`, `"sdl_dialogs.h"`, `"sdl_widgets.h"`
- **External functions (defined elsewhere):**
  - `update_game_window()` ΓÇô refresh game display after alert dismissal
  - `SDL_GetVideoSurface()` ΓÇô check SDL video state
  - `getcstr()` ΓÇô resource string lookup
  - `text_width()` ΓÇô measure text in pixels
  - `get_theme_font()` ΓÇô retrieve themed font and style
  - `stop_recording()` ΓÇô flush game recording/replay state before fatal error
  - `dialog::run()`, `dialog::set_widget_placer()`, `dialog::activate_widget()` ΓÇô dialog modal loop
  - `csprintf()` ΓÇô formatted string buffer helper
  - `exit()`, `abort()` ΓÇô process termination

# Source_Files/CSeries/cscluts.h
## File Purpose
Defines color lookup table (CLUT) data structures and provides functions for building and managing color tables and system colors in the Aleph One game engine. Serves as the interface between the engine's color management system and platform-specific color implementations.

## Core Responsibilities
- Define color value structures (individual RGB colors and lookup tables)
- Declare platform-specific color table construction (`build_macintosh_color_table`)
- Declare portable color table construction (`build_color_table`)
- Enumerate and export system color constants
- Provide predefined color constants (black, white)

## External Dependencies
- **Includes:** `cstypes.h` (for `uint16`, `int16` types)
- **Forward declare:** `LoadedResource` class
- **Defined elsewhere:** `RGBColor` type (platform-specific, likely QuickDraw on Mac or SDL color struct on other platforms); implementations of `build_macintosh_color_table` and `build_color_table`

# Source_Files/CSeries/cscluts_sdl.cpp
## File Purpose
SDL-based implementation for converting Mac CLUT (Color Look-Up Table) resources to internal color_table structures. Provides global color constants used throughout the engine.

## Core Responsibilities
- Convert Mac CLUT resource binary format to color_table structures
- Define and manage global color constants (black, white, system palette)
- Handle endian-aware color data parsing from resource streams

## External Dependencies
- **SDL:** SDL_endian.h, SDL_RWops API (stream operations, endian-aware reads)
- **FileHandler.h:** LoadedResource class
- **cseries.h:** RGBColor definition, cscluts.h include (color_table/rgb_color definitions not visible in provided headers)

# Source_Files/CSeries/csdialogs.h
## File Purpose
Cross-platform dialog box abstraction header providing a unified API for manipulating UI controls on both Mac OS Toolbox and SDL-based implementations. Bridges platform-specific differences in dialog handling and control numbering conventions between Mac and non-Mac platforms.

## Core Responsibilities
- Define platform-agnostic dialog pointer types (DialogPtr, DialogPTR) with conditional compilation
- Declare common cross-platform control manipulation functions (QQ_* functions)
- Provide specialized control-type wrappers (boolean, selector/popup, text fields)
- Define control activity state constants (CONTROL_INACTIVE, CONTROL_ACTIVE)
- Offer Mac OS-specific dialog utilities (event filtering, window frame manipulation, control modification)
- Handle differences in control indexing between Mac OS (1-based) and SDL implementations (0-based)

## External Dependencies
- **Includes**: `<string>` (C++ standard), `<vector>` (C++ standard)
- **Conditional Mac includes**: `<Dialogs.h>` (Mac OS Toolbox, included via SDL_RFORK_HACK macro workaround)
- **Platform detection**: Preprocessor flags `mac`, `SDL_RFORK_HACK`, `USES_NIBS`, `TARGET_API_MAC_CARBON` determine which declarations/types are active
- **Defined elsewhere**: `dialog` class (SDL implementation), `ListHandle`, `ControlHandle`, `WindowRef`, `WindowPtr`, `EventRecord`, `Rect`, `DialogPtr` (native Mac types), various UPP types (Mac toolbox)

# Source_Files/CSeries/csdialogs_sdl.cpp
## File Purpose
Provides cross-platform dialog control compatibility functions between SDL and traditional Mac versions. Implements wrapper functions for querying and manipulating dialog widgets (toggles, selectors, text entries, number entries) with consistent interfaces.

## Core Responsibilities
- Hide/show dialog items by toggling widget enable/disable state
- Extract and insert numeric values from/into text entry widgets
- Query and modify selection control values (w_select widgets)
- Query and modify boolean control values (w_toggle widgets)
- Convert text between C-strings and Pascal-strings (pstrings) for text fields and static text
- Manage widget enable/disable state generically
- Provide null-safe wrapper functions (QQ_* family) that fail gracefully

## External Dependencies
- `cseries.h` ΓÇö Base engine definitions (types, macros)
- `sdl_dialogs.h` ΓÇö Dialog class definition; provides `DialogPtr`, `dialog::get_widget_by_id()`
- `sdl_widgets.h` ΓÇö Widget class hierarchy (w_number_entry, w_select, w_toggle, w_text_entry, w_static_text, w_select_popup)
- C standard library ΓÇö `<string.h>` functions (strlen, strncpy, strcpy)
- Game-local string functions ΓÇö `pstrdup()`, `a1_c2pstr()`, `a1_p2cstr()` (PascalΓåöC conversion); `build_stringvector_from_stringset()` (defined elsewhere)

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

# Source_Files/CSeries/csfiles.h
## File Purpose
Header file declaring file specification and filesystem utility functions for the CSeries compatibility layer. Provides abstraction for classic macOS file operations (FSSpec-based) used throughout the engine.

## Core Responsibilities
- Declare file specification retrieval from resource lists
- Declare application-local file specification acquisition
- Provide macOS filesystem abstraction interface

## External Dependencies
- `OSErr` ΓÇö macOS classic error type (defined elsewhere)
- `FSSpec` ΓÇö classic macOS file specification structure (defined elsewhere)
- License: GNU GPL v2+; copyright Bo Lindbergh and Aleph One contributors

# Source_Files/CSeries/csfiles_beos.cpp
## File Purpose
BeOS-specific utility module providing directory discovery and resource fork handling. Enables reading Marathon data files from Mac CD-ROMs via BeOS file attributes (MACOS:RFORK), and exposes them through SDL_RWops streaming interface for cross-platform asset loading.

## Core Responsibilities
- Locate application and preferences directories on BeOS using native APIs
- Detect and validate resource fork attributes in BeOS file system
- Implement SDL_RWops callbacks (seek, read, write, close) for resource fork attribute access
- Manage file descriptor lifecycle and memory for resource fork streaming
- Support read/write modes for resource fork data

## External Dependencies
- **SDL:** `SDL_rwops.h`, `SDL_error.h` (I/O abstraction layer)
- **BeOS:** `AppKit.h`, `StorageKit.h` (application info, directory finding, path utilities)
- **POSIX:** `unistd.h`, `fcntl.h` (open, close, file control)
- **BeOS filesystem:** `fs_attr.h` (attribute read/write/stat APIs)
- **Standard:** `<string>` (C++ string class)

# Source_Files/CSeries/csfonts.h
## File Purpose
Header file defining font styling constants and text rendering configuration structures for the Aleph One game engine. Provides a portable abstraction for font specifications across platforms, including support for multiple font variants (normal, bold, oblique, bold-oblique) and text styling effects.

## Core Responsibilities
- Define style bit-flags for text rendering (bold, italic, underline, shadow, outline)
- Provide `TextSpec` struct for bundling font configuration (ID, size, style, path data)
- Support per-style font paths for TrueType font rendering
- Enable height adjustment for fine-tuning text layout

## External Dependencies
- **cstypes.h**: Provides platform-specific integer types (`int16`, `uint16`)
- **\<string\>**: Standard library for `std::string` path storage in `TextSpec`


# Source_Files/CSeries/csmacros.h
## File Purpose
Header file providing utility macros and template functions for the Aleph One game engine. Supplies common mathematical operations, bit manipulation helpers, memory management wrappers, and bounds-checked array accessΓÇödesigned to reduce boilerplate and improve type safety across the codebase.

## Core Responsibilities
- Math utilities: min, max, floor, ceiling, clamping, absolute value, sign extraction
- Bit manipulation: flag operations on 32-bit and 16-bit integers
- Generic swap template for any type
- Rectangle dimension calculation helpers
- Power-of-two computation
- Bounds-checked array access with bounds validation
- Type-safe wrappers around `memcpy()` and `memset()` to eliminate explicit `sizeof()` calls

## External Dependencies
- `<string.h>`: `memcpy()`, `memset()`

# Source_Files/CSeries/csmisc.h
## File Purpose
Header file providing platform-specific timing constants and low-level utility functions for the Aleph One game engine. Declares functions for machine tick counting, user input waiting, 68k-specific register access, and system management (screen saver, debugger).

## Core Responsibilities
- Define platform-specific machine tick rates (`MACHINE_TICKS_PER_SECOND`)
- Declare timer/timing functions for the game loop
- Declare user input polling functions
- Provide 68k Motorola CPU register access (inline assembly wrappers for `a0`, `a5`)
- Declare system-level utilities (screen saver management, debugger control)
- Handle platform-conditional compilation (68k Mac, SDL, etc.)

## External Dependencies
- **Preprocessor**: Conditional on `mac`, `SDL`, `env68k`, and `DEBUG` defines for platform/architecture detection.
- **Types used**: `uint32`, `long`, `bool` (defined elsewhere, likely platform abstraction layer).
- **All functions defined elsewhere**: This is a pure declaration header; implementations are in corresponding `.c` files or platform-specific modules.

# Source_Files/CSeries/csmisc_sdl.cpp
## File Purpose
SDL implementation of miscellaneous utility functions for the Aleph One game engine. Provides cross-platform abstractions for tick counting and blocking input waits, isolating SDL-specific code from the rest of the engine.

## Core Responsibilities
- Retrieve high-resolution millisecond tick counter for timing measurements
- Block execution and poll for user input (mouse click or keypress) within a timeout window
- Abstract SDL event loop details behind simple C-like function interfaces

## External Dependencies
- `<SDL.h>` ΓÇô via `cseries.h` include
- `SDL_GetTicks()`, `SDL_PollEvent()`, `SDL_Delay()` ΓÇô external SDL library functions
- `uint32` type ΓÇô defined in `cstypes.h` (included via `cseries.h`)

# Source_Files/CSeries/cspixels.h
## File Purpose
Defines pixel type aliases and bitwise conversion macros for RGB color values in 16-bit and 32-bit pixel formats. Provides low-level support for multi-format pixel manipulation in the Aleph One game engine.

## Core Responsibilities
- Define typed aliases for pixel values at different color depths (8, 16, 32-bit)
- Establish maximum color component values for each pixel format
- Provide macros to pack RGB values (0x0000ΓÇô0xFFFF range) into pixel formats
- Provide macros to extract individual color channels (R, G, B) from packed pixels
- Document color component ranges and format specifications

## External Dependencies
- `cstypes.h` ΓÇö provides uint8, uint16, uint32 type definitions (platform-specific)

---

**Observations:**
- **16-bit format:** The 5:5:5 layout wastes 1 high bit per channel; designed for older 16-bit graphics subsystems.
- **Input/output mismatch:** Combiner macros accept 16-bit values but only use the high 5 bits; extractors return 5-bit or 8-bit values. Caller must scale appropriately if round-tripping is needed.
- **No bounds checking:** Macros assume valid input ranges; overflow silently clips/wraps.

# Source_Files/CSeries/csstrings.cpp
## File Purpose
Provides string manipulation utilities for the Aleph One game engine, supporting legacy Pascal strings (length-prefixed), C strings (null-terminated), and character encoding conversions between Mac Roman and Unicode/UTF-8. Bridges modern C++ string APIs with historical macOS resource-based string systems.

## Core Responsibilities
- Retrieve and manage Pascal/C strings from a resource system (TextStrings)
- Convert between Pascal strings, C strings, and modern `std::string`
- Format strings using printf-style variadic functions
- Perform Mac Roman Γåö Unicode character encoding conversions
- Support UTF-8 encoding/decoding for international text
- Provide safe string operations (bounds-checked copying, in-place conversion)

## External Dependencies
- **TextStrings.h:** `TS_GetString`, `TS_GetCString`, `TS_CountStrings` ΓÇö resource string lookup
- **Logging.h:** `GetCurrentLogger`, `logDomain`, `logAnomalyLevel` ΓÇö logging framework (dprintf/fdprintf now use this)
- **Standard library:** `<stdio.h>`, `<stdarg.h>`, `<string.h>`, `<map>`, `<string>`, `<vector>`
- **Conditional:** `<Carbon/Carbon.h>` when `EXPLICIT_CARBON_HEADER` is defined (macOS legacy support)

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

## External Dependencies
- **Includes**: `cstypes.h` (typed integers), `<string>`, `<vector>` (STL)
- **Compiler attribute**: `PRINTF_STYLE_ARGS` macro (GCC format attribute for type checking)
- **Defined elsewhere**: Resource manager (provides `resid` lookup), Mac Roman encoding tables (for char conversion)
- **Notes**: Implicit dependency on a resource system (Classic Mac-like string resources by ID)

# Source_Files/CSeries/cstypes.h
## File Purpose
Portable type definitions and fixed-point arithmetic infrastructure for the Aleph One game engine. Abstracts platform-specific integer types (Mac/Carbon, BeOS, SDL) into consistent C types and provides fixed-point (16.16) math macros and utility constants.

## Core Responsibilities
- Define portable fixed-width integer types (`int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`) across Mac/BeOS/SDL platforms
- Declare `_fixed` type and fixed-point arithmetic macros (`FIXED_ONE`, `FIXED_INTEGERAL_PART`, etc.)
- Provide min/max constants for integer types (`INT16_MAX`, `INT32_MIN`, etc.)
- Define utility constants (`MEG`, `KILO`) and macro helpers (`FOUR_CHARS_TO_INT`)
- Handle platform-conditional includes and feature detection (OpenGL, config.h)

## External Dependencies
- **Conditional includes:**
  - `<Carbon/Carbon.h>` (Mac with EXPLICIT_CARBON_HEADER)
  - `<support/SupportDefs.h>` (BeOS)
  - `<SDL_types.h>`, `<time.h>` (SDL)
- **Always included:** `<limits.h>`, `config.h` (if HAVE_CONFIG_H)
- **Fallback:** Defines `GLfloat` as `float` if `HAVE_OPENGL` is not defined


# Source_Files/CSeries/mytm.h
## File Purpose
Declares a timer/task management API for scheduling callback functions to execute at specified intervals. Provides thread-safe lifecycle management for timer tasks with mutex protection for multi-threaded contexts (particularly packet listening threads). Part of the Aleph One game engine infrastructure.

## Core Responsibilities
- Schedule periodic timer tasks with callback functions
- Manage task lifecycle (setup, removal, reset)
- Provide mutex locking/unlocking for concurrent access
- Clean up zombie threads and reclaim storage
- Initialize the timer management system

## External Dependencies
- Standard C library (bool type implies C99 or stdbool.h)
- Pthread or platform-specific threading (implied by mutex and thread cleanup)
- Game loop / event system (tasks are scheduled callbacks)

**Defined elsewhere**: `myTMTask` struct implementation, mutex implementation, thread pool/scheduler.

# Source_Files/CSeries/mytm_sdl.cpp
## File Purpose
Provides SDL-based Time Manager emulation for periodic task scheduling. Implements drift-free periodic callbacks using SDL threads, approximating classic Mac OS 9 Time Manager behavior for the Aleph One game engine (especially for networking code). Tasks execute with mutual exclusion and priority boosting to prevent starvation.

## Core Responsibilities
- Initialize and manage a global mutex for synchronizing TMTask execution
- Create and manage periodic callback threads with configurable periods
- Implement drift-free scheduling via cumulative drift tracking across resets
- Handle task lifecycle: setup, execution, reset, and removal
- Support task cleanup and zombie thread collection
- (DEBUG) Track profiling metrics: call counts, timing, drift min/max, late deadlines
- Expose mutex lock/unlock for external code (e.g., packet listening thread)

## External Dependencies
- **SDL_thread.h:** `SDL_Thread`, `SDL_CreateThread()`, `SDL_WaitThread()`
- **SDL_timer.h:** `SDL_GetTicks()`, `SDL_Delay()`
- **SDL_error.h:** `SDL_GetError()`
- **thread_priority_sdl.h:** `BoostThreadPriority()`
- **Logging.h:** `logWarning()`, `logAnomaly*()`, `logDump*()` (variadic logging macros)
- **cseries.h, mytm.h:** Header declarations and macros; `obj_clear()` (trivial helper, likely defined in cseries)
- **std::vector:** Contained in `<vector>` via `using std::vector` (guarded by `!NO_STD_NAMESPACE`)

# Source_Files/CSeries/snprintf.cpp
## File Purpose
Provides portable fallback implementations of `snprintf()` and `vsnprintf()` for platforms that lack them. Acts as a compatibility layer that wraps `vsprintf()` and detects buffer overflows with logging warnings.

## Core Responsibilities
- Provide standards-compliant `snprintf()` wrapper via variadic argument delegation
- Implement `vsnprintf()` with post-hoc overflow detection and logging
- Conditionally compile only on platforms without native implementations
- Prevent recursive warning logs during overflow detection

## External Dependencies
- **Includes:** `<stdio.h>` (vsprintf assumed available), `Logging.h` (logWarning2)
- **Defined elsewhere:** `vsprintf()` (C standard library), `logWarning2()` (Logging subsystem)
- **Conditional directives:** `HAVE_SNPRINTF`, `HAVE_VSNPRINTF` control compilation

# Source_Files/CSeries/snprintf.h
## File Purpose
Compatibility shim header that conditionally declares `snprintf()` and `vsnprintf()` functions. Provides fallback function signatures for platforms lacking C99 standard library implementations, with build-time detection via `config.h`.

## Core Responsibilities
- Conditionally declare `snprintf()` for platforms without native support
- Conditionally declare `vsnprintf()` for platforms without native support
- Gate declarations on `HAVE_SNPRINTF` and `HAVE_VSNPRINTF` autoconf flags
- Provide portable printf-family formatting interface across build targets
- Include MSVC7 workaround (`<streambuf>`)

## External Dependencies
- `<stdarg.h>` ΓÇö va_list type (C standard library)
- `<streambuf>` ΓÇö MSVC7 strangeness workaround (C++ header, unusual in C codebase)
- `"config.h"` ΓÇö autoconf build-time feature detection (defines `HAVE_SNPRINTF`, `HAVE_VSNPRINTF`)

---

**Notes on context (from bundled `config.h`):**  
This is part of **Aleph One** (Marathon-engine game port to SDL, v20080721). Build configuration shows `HAVE_SNPRINTF` and `HAVE_VSNPRINTF` both defined (target has native support), so both conditional declarations are **skipped**. Header is defensive for older/non-POSIX platforms.


