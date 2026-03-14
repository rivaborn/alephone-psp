# Subsystem Overview

## Purpose
CSeries is a cross-platform compatibility library that abstracts platform-specific APIs, data types, and UI paradigms to enable Aleph One to run uniformly across macOS (Classic/Carbon), SDL-based platforms, BeOS, and embedded systems. It provides portable interfaces for type definitions, string handling, pixel operations, file I/O, alerts, dialogs, timing, color management, and endianness conversion.

## Key Files
| File | Role |
|------|------|
| **cseries.h** | Master aggregation header; includes all CSeries subsystems and performs platform detection (macOS, SDL, BeOS, MSVC, GCC) |
| **cstypes.h** | Portable fixed-width integer types (int8/16/32, uint8/16/32) and fixed-point (16.16) arithmetic infrastructure |
| **csstrings.h/cpp** | Pascal/C string conversions, printf-style formatting, Mac RomanΓåöUTF-8 encoding, resource-based string loading |
| **csfonts.h** | Font styling constants (bold, italic, underline, shadow, outline) and TextSpec configuration for TrueType rendering |
| **cspixels.h** | Pixel type aliases (8/16/32-bit) and RGB packing/extraction macros for color depth conversions |
| **cscluts.h/cscluts_sdl.cpp** | Color lookup table (CLUT) structures and Mac CLUT resource conversion to internal color_table format |
| **csalerts.h/csalerts_sdl.cpp** | Alert/error display with severity levels, multi-line text wrapping, stderr fallback, and assertion/warning facilities |
| **csdialogs.h/csdialogs_sdl.cpp** | Dialog box abstraction with cross-platform control manipulation (toggles, text entry, selectors); bridges Mac/SDL control indexing differences |
| **csmacros.h** | Utility macros: min/max, bit operations, power-of-two, bounds-checked array access, memory wrappers (memcpy/memset) |
| **csmisc.h/csmisc_sdl.cpp** | Machine tick counting, high-resolution timing, user input polling, and platform-specific timing constants |
| **byte_swapping.h/cpp** | Endianness-aware byte swapping (16/32-bit fields) for little-endian platforms during game data I/O |
| **csfiles.h/csfiles_beos.cpp** | File specification abstraction and BeOS resource fork handling via SDL_RWops streaming |
| **mytm.h/mytm_sdl.cpp** | Timer task scheduling API with thread-safe lifecycle management, drift-free periodic callbacks, and mutex protection |
| **snprintf.h/cpp** | Portable C99 fallback implementations of snprintf() and vsnprintf() with overflow detection and logging |

## Core Responsibilities
- Aggregate and conditionally expose platform-specific headers (Carbon, Quickdraw, BeOS SupportDefs, SDL) behind portable type names and macros
- Define portable fixed-width integer types and fixed-point arithmetic (16.16 format) with min/max constants
- Convert between Pascal strings, C strings, and std::string with Mac RomanΓåöUTF-8 encoding support via resource-based lookup
- Manipulate dialog controls uniformly across Mac OS Toolbox and SDL implementations, accounting for 1-based vs. 0-based indexing
- Display modal alert dialogs with word-wrapping, fallback stderr logging, and file/line assertion context
- Perform endianness-aware byte swapping on multi-byte fields during binary data loading
- Provide high-resolution tick counting and non-blocking input waits for game loop synchronization
- Schedule periodic callback tasks with thread-safe mutex protection and drift-free cumulative time tracking
- Convert RGB color values to/from 16-bit and 32-bit pixel formats with channel extraction macros
- Supply unified math/bit manipulation macros (min/max, clamping, bit flags, power-of-two) to reduce boilerplate

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- Type definitions: `uint8`, `uint16`, `uint32`, `int8`, `int16`, `int32`, `_fixed` (16.16 fixed-point)
- Pixel/color API: `pixel_t` (8/16/32-bit aliases), `RGBColor`, `color_table`, RGB packing/extraction macros
- String API: `getcstr()`, `TS_GetString()`, `csprintf()`, Mac RomanΓåöUTF-8 conversion functions, `pstrdup()`
- Dialog API: `QQ_*` control manipulation, `DialogPtr`, dialog widget getters/setters (toggle, text, selector values)
- Alert API: `alert()`, `pause_debug()`, assertion/warning macros with optional messages
- Timing API: `machine_tick_count()`, timer task setup/removal, mutex lock/unlock, `obj_clear()` helpers
- File API: `FSSpec`, resource fork SDL_RWops streaming (BeOS-specific)
- Utility: math macros (MIN, MAX, FLOOR, CEILING), bit operations, bounds-checked array access

**Consumes from other subsystems:**
- **Logging.h**: `logWarning()`, `logAnomaly()`, `logDump()` (for overflow warnings and task profiling)
- **Rendering/UI**: `update_game_window()` (alert dismissal refresh), `text_width()`, `get_theme_font()` (text layout)
- **Recording/Replay**: `stop_recording()` (flush state before fatal errors)
- **Dialog/Widget framework**: `dialog::run()`, `dialog::set_widget_placer()`, widget enable/disable (SDL implementation details)
- **Resource system**: `TextStrings` resource manager, `LoadedResource` class (asset loading)
- **Platform SDK**: SDL (event, threading, timer), Mac Toolbox (Dialogs, Controls, Events), BeOS (AppKit, StorageKit, fs_attr)

## Runtime Role
- **Init phase**: `cseries.h` conditionally includes platform headers at compile time; `mytm_sdl.cpp` initializes global mutex and timer task infrastructure; `cstypes.h` sets up fixed-point constants
- **Frame loop**: `csmisc_sdl::machine_tick_count()` polled for frame timing; timer tasks scheduled in `mytm_sdl.cpp` execute callbacks at drift-corrected intervals; `SDL_GetTicks()` advances cumulative drift tracking
- **User input**: `csmisc_sdl::wait_for_input()` blocks game loop on keypress/mouse click within timeout
- **Shutdown**: `mytm_sdl` performs task cleanup and zombie thread collection; dialogs/alerts torn down via platform-specific cleanup (implicit in SDL/Mac resource deallocation)
- **On-demand**: String conversions (csstrings.cpp) execute during load; alert display (csalerts_sdl.cpp) blocks rendering loop; byte swapping (byte_swapping.cpp) applied during binary data deserialization

## Notable Implementation Details
- **Fixed-point arithmetic (16.16 format)**: `FIXED_ONE` = 0x10000; integer part extracted via right-shift; enables precise sub-pixel calculations without floating-point overhead on embedded systems (PSP VFPU can optimize this)
- **Drift-free task scheduling (mytm_sdl.cpp)**: Cumulative drift tracked across resets; periodic callback thread sleeps adjusted to account for prior execution time; prevents timer drift from accumulating over long gameplay sessions
- **PascalΓåöC string bridge (csstrings.cpp)**: Length-prefixed Mac-era strings converted to/from modern std::string via intermediate C buffers; `a1_p2cstr()`, `a1_c2pstr()` perform in-place conversions
- **Alert text word-wrapping (csalerts_sdl.cpp)**: Multi-line wrapping to `MAX_ALERT_WIDTH` pixels using `text_width()` measurement; falls back to stderr when `SDL_GetVideoSurface()` unavailable (headless mode)
- **Bounds-checked array macros (csmacros.h)**: Template-based access validation prevents out-of-bounds reads; `bounds_check<T>()` asserts index within range
- **Platform-conditional compilation gates (cseries.h, byte_swapping.cpp)**: Preprocessor directives (`__PSP__`, `ALEPHONE_LITTLE_ENDIAN`, `SDL`, `__APPLE__`) control which headers/implementations compile; enables single codebase across disparate targets
- **Per-style font paths (csfonts.h TextSpec)**: Supports separate TrueType font files for bold/italic/outline variants; enables high-quality text rendering without font rasterization
- **Control indexing reconciliation (csdialogs_sdl.cpp)**: Wrapper functions (`QQ_*` family) hide 1-based Mac OS (DialogItemIndex) vs. 0-based SDL (widget ID) indexing; callers use unified interface
