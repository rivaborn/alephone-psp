# Source_Files/CSeries/csfiles_beos.cpp

## File Purpose
BeOS-specific utility module providing directory discovery and resource fork handling. Enables reading Marathon data files from Mac CD-ROMs via BeOS file attributes (MACOS:RFORK), and exposes them through SDL_RWops streaming interface for cross-platform asset loading.

## Core Responsibilities
- Locate application and preferences directories on BeOS using native APIs
- Detect and validate resource fork attributes in BeOS file system
- Implement SDL_RWops callbacks (seek, read, write, close) for resource fork attribute access
- Manage file descriptor lifecycle and memory for resource fork streaming
- Support read/write modes for resource fork data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rfork_data` | struct | Tracks file descriptor, current read/write position, and attribute size for active resource fork stream |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ATTR_NAME` | const char* | file-static | BeOS attribute name for Mac resource forks: `"MACOS:RFORK"` |

## Key Functions / Methods

### get_application_directory
- **Signature:** `string get_application_directory(void)`
- **Purpose:** Retrieve the directory containing the running application executable
- **Inputs:** None
- **Outputs/Return:** String path to application directory
- **Side effects:** Queries BeOS app_info through global `be_app`
- **Calls:** `be_app->GetAppInfo()`, `BEntry`, `BPath`, `BPath::GetParent()`
- **Notes:** Used to locate bundled data/resources relative to executable location

### get_preferences_directory
- **Signature:** `string get_preferences_directory(void)`
- **Purpose:** Retrieve the user settings directory for Aleph One on BeOS
- **Inputs:** None
- **Outputs/Return:** String path to `B_USER_SETTINGS_DIRECTORY/Aleph One`
- **Side effects:** Creates "Aleph One" subdirectory if not present (third arg `true`)
- **Calls:** `find_directory()`, `BPath::Append()`, `BPath::Path()`
- **Notes:** Standard BeOS convention for per-application user settings

### has_rfork_attribute
- **Signature:** `bool has_rfork_attribute(const char *file)`
- **Purpose:** Check whether a file has a MACOS:RFORK attribute readable on BeOS
- **Inputs:** `file` ΓÇô file path to check
- **Outputs/Return:** `true` if attribute exists, `false` otherwise or on error
- **Side effects:** Opens/closes file descriptor
- **Calls:** `open()`, `fs_stat_attr()`, `close()`
- **Notes:** Non-blocking check; handles open failure gracefully

### sdl_rw_from_rfork
- **Signature:** `SDL_RWops *sdl_rw_from_rfork(const char *file, bool writable)`
- **Purpose:** Create an SDL_RWops stream interface for reading/writing a resource fork attribute
- **Inputs:** `file` ΓÇô path to file with RFORK attribute; `writable` ΓÇô open mode (O_RDWR vs O_RDONLY)
- **Outputs/Return:** Allocated SDL_RWops pointer with callbacks installed; NULL on any failure
- **Side effects:** Allocates SDL_RWops and rfork_data; opens file descriptor; caller must invoke `close()` callback to clean up
- **Calls:** `open()`, `fs_stat_attr()`, `SDL_AllocRW()`, `malloc()`, `close()`, `free()`
- **Notes:** Multi-level error handling with cascading cleanup; fd and data pointers stored in `rwops->hidden.unknown.data1`

### rfork_seek / rfork_read / rfork_write / rfork_close
- **Collective Purpose:** SDL_RWops callback implementations for resource fork attribute access
- **Notes:** `rfork_seek` clamps bounds to [0, size]; `rfork_read`/`rfork_write` clamp element count to available space; `rfork_close` safely handles NULL context and frees all allocated resources

## Control Flow Notes
Initialization phase: `get_application_directory()` and `get_preferences_directory()` are called during engine startup to locate data and settings. Asset loading phase: when engine encounters Mac CD-ROM files with resource forks, `has_rfork_attribute()` + `sdl_rw_from_rfork()` enable transparent streaming of fork data into the engine's asset loaders. The SDL_RWops abstraction decouples BeOS-specific attribute I/O from platform-independent game code.

## External Dependencies
- **SDL:** `SDL_rwops.h`, `SDL_error.h` (I/O abstraction layer)
- **BeOS:** `AppKit.h`, `StorageKit.h` (application info, directory finding, path utilities)
- **POSIX:** `unistd.h`, `fcntl.h` (open, close, file control)
- **BeOS filesystem:** `fs_attr.h` (attribute read/write/stat APIs)
- **Standard:** `<string>` (C++ string class)
