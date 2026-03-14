# Source_Files/Network/Update.h

## File Purpose
Declares a singleton `Update` class that checks for new versions of the Aleph One game engine asynchronously. Manages update-check status and new version information, running the check in a background SDL thread to avoid blocking the main application.

## Core Responsibilities
- Singleton instance management for centralized update checking
- Status tracking (checking, failed, available, not available)
- Storage of new version metadata (date version, display version)
- Background thread management for non-blocking update checks
- Public interface for querying current status and available version

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Status` | enum | Update check state: CheckingForUpdate, UpdateCheckFailed, UpdateAvailable, NoUpdateAvailable |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Update*` | static | Singleton instance pointer |
| `m_status` | `Status` | instance | Current update check status |
| `m_new_date_version` | `std::string` | instance | Version string (date-based format) |
| `m_new_display_version` | `std::string` | instance | Human-readable version string |
| `m_thread` | `SDL_Thread*` | instance | Handle to background thread |

## Key Functions / Methods

### instance()
- Signature: `static Update *instance()`
- Purpose: Lazy-initialize and return singleton instance
- Outputs/Return: Pointer to the single Update object
- Side effects: Creates instance on first call
- Notes: Implements classic C++ singleton pattern

### GetStatus()
- Signature: `Status GetStatus()`
- Purpose: Query current update-check state
- Outputs/Return: Current `Status` enum value
- Notes: Non-blocking, returns immediately

### NewDisplayVersion()
- Signature: `std::string NewDisplayVersion()`
- Purpose: Retrieve formatted version string for user display
- Outputs/Return: Human-readable version (e.g., "1.2.3")
- Notes: Asserts `m_status == UpdateAvailable`; undefined behavior if called when update unavailable

### StartUpdateCheck()
- Signature: `void StartUpdateCheck()` (private)
- Purpose: Initiates background update check thread
- Side effects: Spawns `m_thread`, modifies `m_status`

### update_thread() / Thread()
- Signature: `static int update_thread(void*)` and `int Thread()` (private)
- Purpose: Entry point and execution function for background thread
- Side effects: Network I/O, updates `m_status` and version strings
- Notes: `update_thread` wraps `Thread()` (SDL thread callback pattern)

## Control Flow Notes
Typical flow:
1. Application calls `Update::instance()` to initialize
2. At startup, `StartUpdateCheck()` is likely called to spawn background thread
3. Thread runs `Thread()`, performs network request, updates `m_status` and version strings
4. Main thread polls `GetStatus()` periodically or responds to completion
5. If `UpdateAvailable`, application displays `NewDisplayVersion()`

The design prevents blocking the main render/input loop by deferring update checks to a separate thread.

## External Dependencies
- `<SDL_thread.h>` / `<SDL/SDL_thread.h>` ΓÇô Cross-platform threading primitives
- `<string>` ΓÇô Standard C++ string storage
- `cseries.h` ΓÇô Project's common cross-platform utilities (likely includes assert macros, SDL setup)
