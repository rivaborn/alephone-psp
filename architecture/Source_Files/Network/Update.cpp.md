# Source_Files/Network/Update.cpp

## File Purpose
Implements an online update checker for Aleph One that runs in a background thread. Connects to a remote server, retrieves the latest version information for the current platform, and reports whether an update is available by comparing versions.

## Core Responsibilities
- Manages the update-check singleton instance and its lifecycle
- Spawns and joins SDL worker threads for non-blocking network operations
- Establishes TCP connections and sends HTTP GET requests to the update server
- Parses HTTP response headers to extract version metadata
- Compares remote version (date-based string) against the local version constant
- Tracks update check status (checking, failed, available, or not available)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Status (enum) | enum | Tracks update check states (CheckingForUpdate, UpdateCheckFailed, UpdateAvailable, NoUpdateAvailable) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Update*` | static singleton | Single global instance of the update checker |

## Key Functions / Methods

### Constructor: `Update()`
- Signature: `Update()`
- Purpose: Initialize the update checker and start the first check
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `m_status` to `NoUpdateAvailable`, spawns a worker thread via `StartUpdateCheck()`
- Calls: `StartUpdateCheck()`
- Notes: Singleton pattern enforced by private constructor

### Destructor: `~Update()`
- Signature: `~Update()`
- Purpose: Clean up the worker thread if one exists
- Inputs: None
- Outputs/Return: None
- Side effects: Waits for `m_thread` to complete using `SDL_WaitThread()`
- Calls: `SDL_WaitThread()`

### `StartUpdateCheck()`
- Signature: `void StartUpdateCheck()`
- Purpose: Initiate a new update check, preventing concurrent checks
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `m_status` to `CheckingForUpdate`, clears `m_new_date_version` and `m_new_display_version`, creates a new SDL thread
- Calls: `SDL_CreateThread()`, `SDL_WaitThread()`
- Notes: Early return if check already in progress; waits for prior thread before creating a new one

### `update_thread()` (static)
- Signature: `static int update_thread(void *p)`
- Purpose: Thread entry point wrapper that casts void* to Update* and delegates to `Thread()`
- Inputs: `void *p` (Update* cast to void*)
- Outputs/Return: Return value from `Thread()`
- Side effects: None (just delegation)
- Calls: Implicit call to `Thread()`

### `Thread()`
- Signature: `int Thread()`
- Purpose: Execute the main network operation in the worker thread
- Inputs: None (uses member variables for state)
- Outputs/Return: Return code (0 on success, 1ΓÇô5 on various failure stages)
- Side effects: 
  - Resolves hostname and opens TCP socket
  - Sends HTTP GET request to `marathon.sourceforge.net:80`
  - Receives up to 8192 bytes of response
  - Parses response for `A1_DATE_VERSION:` and `A1_DISPLAY_VERSION:` lines
  - Sets `m_status` and populates `m_new_date_version` / `m_new_display_version`
- Calls: `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()`, string comparison and parsing
- Notes: 
  - No timeout handling; network operations can block indefinitely
  - Minimal error handling; all network errors collapse to `UpdateCheckFailed`
  - Hardcoded server and path; platform-specific via `A1_UPDATE_PLATFORM` macro
  - Uses Boost tokenizer to split response by `\r\n` line endings
  - Version comparison is string-based (`compare() > 0`); assumes date-version format is lexicographically orderable (e.g., "20080721")
  - Known limitations per comment: does not handle HTTP Location headers or chunked transfer encoding

## Control Flow Notes
- **Initialization**: Constructor calls `StartUpdateCheck()`, which spawns a background thread
- **Thread lifecycle**: Worker thread runs `Thread()`, updates `m_status` and version strings, then exits
- **Main-thread polling**: Code elsewhere calls `GetStatus()` and `NewDisplayVersion()` to check results without blocking
- **Singleton pattern**: `instance()` static method ensures only one Update object exists

## External Dependencies
- **SDL_net**: `SDLNet_ResolveHost()`, `SDLNet_TCP_Open()`, `SDLNet_TCP_Send()`, `SDLNet_TCP_Recv()` for networking
- **SDL threads**: `SDL_CreateThread()`, `SDL_WaitThread()` for background execution
- **Boost**: `boost::tokenizer` and `boost::char_separator` for HTTP response parsing
- **alephversion.h**: Defines `A1_DATE_VERSION`, `A1_DISPLAY_VERSION`, `A1_UPDATE_PLATFORM` (platform-specific build macros)
