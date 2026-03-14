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

## Key Types / Data Structures
None. File uses only external types: `dialog`, `vertical_placer`, `w_title`, `w_static_text`, `w_spacer`, `w_button` from sdl_dialogs.h and sdl_widgets.h.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MAX_ALERT_WIDTH` | const int | file-static | Maximum pixel width for alert message text before wrapping (320px) |
| `assert_text` | char[256] | file-static | Formatting buffer for assert/warn messages |

## Key Functions / Methods

### alert_user (const char \*message, short severity)
- **Signature:** `void alert_user(const char *message, short severity)`
- **Purpose:** Display a modal alert/error dialog with word-wrapped text, or fall back to stderr.
- **Inputs:** 
  - `message`: alert text to display
  - `severity`: `infoError` (info) or `fatalError` (fatal)
- **Outputs/Return:** None (void); may exit(1) if fatal.
- **Side effects:** 
  - Calls `SDL_GetVideoSurface()` to check for available video
  - Creates dialog, widgets, and runs modal loop (blocking)
  - Calls `update_game_window()` if info-level and no parent dialog
  - Exits process with code 1 if severity == fatalError
  - Writes to stderr if no video surface
- **Calls:** `SDL_GetVideoSurface()`, `fprintf()`, `text_width()`, `get_theme_font()`, `strdup()`, `free()`, `d.run()`, `update_game_window()`, `exit()`
- **Notes:** 
  - Word-wrapping loop modifies local copy of message string
  - Button label and title change based on severity (OK vs. QUIT)
  - Takes dialog ownership of all created widgets via placer

### alert_user (short severity, short resid, short item, OSErr error)
- **Signature:** `void alert_user(short severity, short resid, short item, OSErr error)`
- **Purpose:** Overloaded variant that retrieves error string from resource, logs it, and delegates to string-based alert_user.
- **Inputs:** 
  - `severity`: `infoError` or `fatalError`
  - `resid`, `item`: resource ID and item for `getcstr()` lookup
  - `error`: OSErr code to include in message
- **Outputs/Return:** None (void).
- **Side effects:** 
  - Calls `logError2()` or `logFatal2()` depending on severity
  - Constructs formatted string with sprintf
  - Delegates to `alert_user(msg, severity)`
- **Calls:** `getcstr()`, `sprintf()`, `logError2()`, `logFatal2()`, `alert_user(const char*, short)`
- **Notes:** Resource ID pattern typical of classic Mac/cross-platform game engines.

### pause_debug
- **Signature:** `void pause_debug(void)`
- **Purpose:** Log a pause/debug breakpoint; no-op on SDL (no true debugger integration).
- **Inputs:** None.
- **Outputs/Return:** None (void).
- **Side effects:** Logs note, prints to stderr.
- **Calls:** `logNote()`, `fprintf()`
- **Notes:** Intended as a debugging hook; does not actually pause or break into debugger.

### vpause
- **Signature:** `void vpause(const char *message)`
- **Purpose:** Log and display a warning message for debugging; non-fatal pause point.
- **Inputs:** `message`: debug warning text.
- **Outputs/Return:** None (void).
- **Side effects:** Logs warning, prints to stderr.
- **Calls:** `logWarning1()`, `fprintf()`
- **Notes:** No UI displayed; stderr only.

### halt
- **Signature:** `void halt(void)`
- **Purpose:** Fatal error halt without message; exit immediately.
- **Inputs:** None.
- **Outputs/Return:** None (void; exits process).
- **Side effects:** Logs fatal, calls `abort()`.
- **Calls:** `logFatal()`, `fprintf()`, `abort()`
- **Notes:** No user message; used for unrecoverable internal errors.

### vhalt
- **Signature:** `void vhalt(const char *message)`
- **Purpose:** Fatal halt with error message; attempts crash dump on macOS, abort on other platforms.
- **Inputs:** `message`: final error message.
- **Outputs/Return:** None (void; exits process).
- **Side effects:** 
  - Calls `stop_recording()` to flush game state
  - Logs fatal message
  - On macOS (\_\_APPLE\_\_ && \_\_MACH\_\_): dereferences NULL pointer (intentional crash for debugger attachment)
  - On other platforms: calls `abort()`
- **Calls:** `stop_recording()`, `logFatal1()`, `fprintf()`, `abort()` or null dereference
- **Notes:** Intentional null dereference on macOS allows post-mortem debugging; other platforms use standard abort().

### _alephone_assert
- **Signature:** `void _alephone_assert(const char *file, long line, const char *what)`
- **Purpose:** Handler for failed assertions; formats and calls vhalt.
- **Inputs:** 
  - `file`: source file name
  - `line`: source line number
  - `what`: assertion condition/message
- **Outputs/Return:** None (void; exits via vhalt).
- **Side effects:** Formats assertion text, calls vhalt (fatal).
- **Calls:** `csprintf()`, `vhalt()`
- **Notes:** Reuses static `assert_text` buffer; typical assert macro integration point.

### _alephone_warn
- **Signature:** `void _alephone_warn(const char *file, long line, const char *what)`
- **Purpose:** Handler for assertion warnings (non-fatal); formats and calls vpause.
- **Inputs:** 
  - `file`: source file name
  - `line`: source line number
  - `what`: assertion condition/message
- **Outputs/Return:** None (void).
- **Side effects:** Formats warning text, calls vpause (non-fatal).
- **Calls:** `csprintf()`, `vpause()`
- **Notes:** Same formatting as _alephone_assert but non-fatal; reuses `assert_text` buffer.

## Control Flow Notes
These functions are **error/alert handlers**, not part of the main game loop. They are called reactively when:
- A fatal error or assertion fails ΓåÆ `_alephone_assert()` ΓåÆ `vhalt()` ΓåÆ exit
- A non-fatal warning fires ΓåÆ `_alephone_warn()` ΓåÆ `vpause()` ΓåÆ continue
- User code wants to show an error dialog ΓåÆ `alert_user()` ΓåÆ modal loop ΓåÆ return (or exit if fatal)

The `alert_user()` dialog is **blocking**: it enters a modal event loop and does not return until the user clicks OK or the alert severity is fatal.

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
