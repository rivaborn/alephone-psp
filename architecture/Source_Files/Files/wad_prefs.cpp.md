# Source_Files/Files/wad_prefs.cpp

## File Purpose
Manages persistent player preferences storage for the Aleph One game engine using WAD (Where's All the Data) file format. Handles initialization, loading, retrieval with lazy initialization/validation, and saving of preferences with automatic error recovery for corrupted files.

## Core Responsibilities
- Initialize and open preferences file (create if missing, delete and recreate if corrupted)
- Load preference WAD from disk into memory
- Retrieve individual preference entries by tag with lazy initialization and optional validation
- Write modified preferences back to disk in WAD format
- Provide callback hooks for preference initialization and validation
- Handle platform-specific file paths (Mac vs. SDL)
- Implement graceful error recovery and reporting

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `preferences_info` | struct | Contains file specifier and in-memory WAD pointer |
| `prefs_initializer` | typedef (function pointer) | Callback to initialize preference when first created |
| `prefs_validater` | typedef (function pointer) | Callback to validate/fix preference data and return whether modified |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `prefInfo` | `struct preferences_info *` | file-static | Singleton holding current prefs file and WAD data |
| `CURRENT_PREF_WADFILE_VERSION` | #define (0) | file-static | WAD format version for preferences |

## Key Functions / Methods

### w_open_preferences_file
- **Signature:** `bool w_open_preferences_file(char *PrefName, Typecode Type)`
- **Purpose:** Entry point; allocates preferences structure and loads file
- **Inputs:** Filename, file type code (platform-specific)
- **Outputs/Return:** `true` if initialized successfully; `false` if fatal error
- **Side effects:** Allocates `prefInfo` on heap; calls `load_preferences()`; on error, may delete/recreate file and initialize empty WAD; sets game errors
- **Calls:** `load_preferences()`, `create_empty_wad()`, `w_write_preferences_file()`, `error_pending()`, `get_game_error()`, `set_game_error()`
- **Notes:** Catches exceptions for memory errors; handles both Mac and SDL file paths conditionally; auto-recovery deletes corrupt files before recreating

### w_get_data_from_preferences
- **Signature:** `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate)`
- **Purpose:** Retrieve preference by tag; create with initializer if missing; fix with validator if corrupted
- **Inputs:** Tag (data type), expected size, init callback, validation callback
- **Outputs/Return:** Pointer to data within WAD (valid until next modification)
- **Side effects:** May allocate temporary memory; appends new/modified data to WAD
- **Calls:** `extract_type_from_wad()`, `malloc()`, `free()`, `append_data_to_wad()`, `memcpy()`
- **Notes:** Handles size mismatches by reinitializing; validator can modify data; makes temporary copy before appending to avoid WAD pointer invalidation

### w_write_preferences_file
- **Signature:** `void w_write_preferences_file(void)`
- **Purpose:** Serialize and persist preferences WAD to disk
- **Inputs:** None (uses `prefInfo`)
- **Outputs/Return:** None
- **Side effects:** Writes WAD header, data, directory to file; deletes old file before writing
- **Calls:** `fill_default_wad_header()`, `open_wad_file_for_writing()`, `write_wad_header()`, `calculate_wad_length()`, `set_indexed_directory_offset_and_length()`, `write_wad()`, `write_directorys()`, `close_wad_file()`
- **Notes:** Safe to call at exit; clears pending errors first; recreates file to avoid MacOS existence issues

### load_preferences
- **Signature:** `static void load_preferences(void)`
- **Purpose:** Load WAD from disk into `prefInfo->wad`
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Frees old WAD if present; opens file and reads header/indexed WAD; may set game errors
- **Calls:** `free_wad()`, `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `set_game_error()`
- **Notes:** Graceful degradation: unknown WAD version sets error but doesn't crash

## Control Flow Notes
Singleton preferences manager in the initialization/shutdown pipeline:
- **Startup:** Engine calls `w_open_preferences_file()` once to load user settings
- **Runtime:** Other systems call `w_get_data_from_preferences()` for individual values (lazy init on first access)
- **Shutdown:** Engine calls `w_write_preferences_file()` (possibly from atexit handler) to persist changes

## External Dependencies
- **Includes:** cseries.h, wad.h, game_errors.h, wad_prefs.h, FileHandler.h, string.h, stdlib.h, stdexcept
- **External symbols:** `create_empty_wad()`, `extract_type_from_wad()`, `append_data_to_wad()`, `free_wad()` (WAD ops); `open_wad_file_for_reading/writing()`, `read_wad_header()`, `write_wad_header()`, `write_directorys()`, `calculate_wad_length()` (file I/O); `set_game_error()`, `error_pending()` (error handling); `MemError()` (Mac); `dprintf()` (debug)
