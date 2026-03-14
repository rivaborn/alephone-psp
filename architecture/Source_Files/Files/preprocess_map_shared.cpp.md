# Source_Files/Files/preprocess_map_shared.cpp

## File Purpose

Implements automatic game-save functionality for the Aleph One engine (Marathon-like game). Provides mechanisms to save games without user interaction, generate filename-friendly strings, and automatically resolve filename conflicts by appending numeric suffixes.

## Core Responsibilities

- Auto-save game state without presenting save dialogs to the user
- Sanitize arbitrary text into filename-safe strings (alphanumeric + spaces only)
- Generate non-conflicting filename variants by appending numeric suffixes (e.g., "Map", "Map 2", "Map 3")
- Report save success/failure to screen via `screen_printf()`
- Support overwriting recent saves or creating new unique filenames based on current level name

## Key Types / Data Structures

None (file-local only).

## Global / File-Static State

| Name | Type | Scope (global/static/singleton) | Purpose |
|------|------|--------------------------------|---------|
| `kMaxFilenameChars` | `enum` | static | Platform-dependent max filename length (31 on Mac, 255 on SDL) |

## Key Functions / Methods

### strncpy_filename_friendly
- **Signature**: `static int32 strncpy_filename_friendly(char* outString, const char* inString, int32 inOutputStringBufferSize)`
- **Purpose**: Sanitizes a string for safe use as a filename by preserving only alphanumeric characters and spaces.
- **Inputs**: `inString` (source C-string), `inOutputStringBufferSize` (output buffer size in bytes)
- **Outputs/Return**: Writes sanitized string to `outString`; returns length of output string (not including null terminator)
- **Side effects**: Fills output buffer with sanitized text; pads remainder with nulls via `memset()`
- **Calls**: `isalnum()`, `memset()`
- **Notes**: Asserts preconditions on non-null inputs and positive buffer size; truncates silently if input is longer than output capacity

### make_nonconflicting_filename_variant
- **Signature**: `static bool make_nonconflicting_filename_variant(FileSpecifier& inBaseName, FileSpecifier& outNonconflictingName, size_t inMaxNameLength)`
- **Purpose**: Generates a unique filename by appending numeric suffixes until an unused variant is found.
- **Inputs**: `inBaseName` (base filename spec), `inMaxNameLength` (max chars allowed in final filename)
- **Outputs/Return**: Sets `outNonconflictingName` on success; returns `true` if a variant was found, `false` otherwise
- **Side effects**: Modifies `outNonconflictingName` FileSpecifier; calls filesystem `Exists()` repeatedly
- **Calls**: `FileSpecifier::ToDirectory()`, `GetName()`, `FromDirectory()`, `AddPart()`, `SetName()`, `Exists()` (platform-specific); `sprintf()`
- **Notes**: Loop attempts variants "Base", "Base 2", "Base 3", etc. Truncates base name if suffix + base exceeds `inMaxNameLength`. Comment notes infinite-loop risk is negligible (would require filesystem containing every possible variant).

### save_game_full_auto
- **Signature**: `bool save_game_full_auto(bool inOverwriteRecent)`
- **Purpose**: Saves the current game state to disk, either overwriting the most recent save or creating a new uniquely-named file.
- **Inputs**: `inOverwriteRecent` (if `true`, overwrite recent save; if `false` or no recent save exists, create new file)
- **Outputs/Return**: Returns `true` if save succeeded, `false` if save failed
- **Side effects**: Writes game data to filesystem via `save_game_file()`; calls `screen_printf()` to report result to player; accesses `static_world->level_name` (global game state)
- **Calls**: `get_current_saved_game_name()`, `strncpy_filename_friendly()`, `strcpy()`, `make_nonconflicting_filename_variant()`, `save_game_file()`, `screen_printf()`, and `FileSpecifier` I/O methods
- **Notes**: Treats default save name as "no recent save"; falls back to "Automatic Save" if level name yields empty or zero-length sanitized name; on success, reports "Created new" or "Replaced existing" with the actual filename used

## Control Flow Notes

**Initialization flow**: NoneΓÇöthis file contains no init/setup code.

**Game state**: Called during gameplay (likely from netgame or auto-save systems). Reads `static_world->level_name` to derive a suggested filename; queries filesystem to find an available variant.

**Shutdown flow**: None.

## External Dependencies

- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier` classes for filesystem abstraction
- **game_wad.h**: `get_current_saved_game_name()`, `save_game_file()` (save/load mechanisms)
- **interface.h**: `TS_GetCString()` (string resource lookup), `screen_printf()` (on-screen messaging)
- **map.h**: `static_world->level_name` (current level name from world state)
- **shell.h**: `screen_printf()` (display function)
- **TextStrings.h**: `TS_GetCString()` (localized string fetching)
- **Standard C**: `<ctype.h>` (`isalnum()`), `<string.h>` implicit (`strcpy()`, `strlen()`, `memset()`)
