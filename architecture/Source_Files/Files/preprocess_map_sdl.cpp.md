# Source_Files/Files/preprocess_map_sdl.cpp

## File Purpose
SDL implementation of save game file handling and default data file localization. Provides functions to locate critical game data files (maps, shapes, sounds, physics models) from a configurable search path, and implements save/load game dialog functionality.

## Core Responsibilities
- Locate default game data files (map, physics model, shapes, sounds, music, theme) from a search path
- Display read/write dialogs for save game selection and creation
- Handle game state persistence (save_game and related game_wad functions)
- Alert user when critical files (map, shapes) cannot be found
- Support graceful degradation when optional files (physics, sounds) are missing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class | Encapsulates file path and operations; from FileHandler.h |
| `DirectorySpecifier` | class | Encapsulates directory path; from FileHandler.h |
| `vector<DirectorySpecifier>` | STL container | Search path for locating data files |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | global (extern) | Data directory search path; defined in shell_sdl.cpp |

## Key Functions / Methods

### get_default_spec (string overload)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, const string &name)`
- **Purpose:** Locate a file by name in the data search path
- **Inputs:** Reference to FileSpecifier (output), filename string
- **Outputs/Return:** Boolean true if file found and exists
- **Side effects:** Modifies the `file` parameter to point to located file
- **Calls:** `FileSpecifier::Exists()`, iterator operations on `data_search_path`
- **Notes:** Iterates linearly through search path; returns on first match

### get_default_spec (int overload)
- **Signature:** `static bool get_default_spec(FileSpecifier &file, int type)`
- **Purpose:** Locate file by string resource ID (resource-driven lookup)
- **Inputs:** Reference to FileSpecifier, resource string ID
- **Outputs/Return:** Boolean success
- **Side effects:** Calls `getcstr()` to retrieve filename from resource, then calls string overload
- **Calls:** `getcstr()`, `get_default_spec(FileSpecifier&, const string&)`
- **Notes:** Abstraction layer for resource-based filenames

### get_default_map_spec
- **Signature:** `void get_default_map_spec(FileSpecifier &file)`
- **Purpose:** Locate the default game map file (critical)
- **Inputs:** Reference to FileSpecifier (output)
- **Outputs/Return:** None (void)
- **Side effects:** Calls `get_default_spec()` with `filenameDEFAULT_MAP`; alerts user if not found
- **Calls:** `get_default_spec(FileSpecifier&, int)`, `alert_user()`
- **Notes:** Fatal error if map cannot be found; blocks game startup

### save_game
- **Signature:** `bool save_game(void)`
- **Purpose:** Initiate game save sequence with user dialog
- **Inputs:** None
- **Outputs/Return:** Boolean success status
- **Side effects:** Pauses game, shows/hides cursor, calls `save_game_file()`, resumes game
- **Calls:** `pause_game()`, `show_cursor()`, `get_current_saved_game_name()`, `file.GetName()`, `getcstr()`, `file.WriteDialogAsync()`, `save_game_file()`, `hide_cursor()`, `resume_game()`
- **Notes:** Async dialog allows sound/music to continue; uses stored current save game name as default

### add_finishing_touches_to_save_file
- **Signature:** `void add_finishing_touches_to_save_file(FileSpecifier &file)`
- **Purpose:** Post-process save file with metadata (currently stubbed)
- **Inputs:** Reference to FileSpecifier for the save file
- **Outputs/Return:** None (void)
- **Side effects:** None in current implementation
- **Calls:** None
- **Notes:** Placeholder for future Mac OS resource fork storage (thumbnail, level name); empty on SDL

### choose_saved_game_to_load
- **Signature:** `bool choose_saved_game_to_load(FileSpecifier &saved_game)`
- **Purpose:** Display file dialog to user for selecting a save game to load
- **Inputs:** Reference to FileSpecifier (output)
- **Outputs/Return:** Boolean success (user selected a file)
- **Side effects:** Modifies `saved_game` parameter if user selects file
- **Calls:** `FileSpecifier::ReadDialog()`
- **Notes:** Delegates directly to FileSpecifier dialog; returns false if user cancels

### get_default_music_spec, get_default_physics_spec, get_default_sounds_spec, get_default_shapes_spec
- **Signature:** Each `void` or `bool get_default_X_spec(FileSpecifier &file)`
- **Purpose:** Locate optional/critical game data files
- **Inputs:** FileSpecifier reference (output)
- **Outputs/Return:** Boolean (music only); others void
- **Side effects:** Modifies file parameter; shapes/map alert on failure; physics/sounds silent fail
- **Calls:** `get_default_spec()`
- **Notes:** Differentiated error handling based on criticality

### get_default_theme_spec
- **Signature:** `bool get_default_theme_spec(FileSpecifier &file)`
- **Purpose:** Locate theme file from "Themes" subdirectory
- **Inputs:** FileSpecifier reference (output)
- **Outputs/Return:** Boolean success
- **Side effects:** Constructs path as "Themes/" + theme filename
- **Calls:** `FileSpecifier()` constructor, `getcstr()`, `get_default_spec()`
- **Notes:** Special handling for theme subdirectory structure

## Control Flow Notes
**Startup/Initialization:** `get_default_map_spec()`, `get_default_shapes_spec()`, and related functions are called during game initialization to ensure critical files are located before entering the game world.

**During Gameplay:** `save_game()` can be called asynchronously from the main loop; `choose_saved_game_to_load()` is called when user requests load from menu.

**File I/O Bridge:** This file acts as a bridge between the game state management (pause_game, resume_game, save_game_file) and the cross-platform file dialog system (FileSpecifier).

## External Dependencies
- **cseries.h** ΓÇö Core types, macros, platform abstractions (vector, string)
- **FileHandler.h** ΓÇö `FileSpecifier` and `DirectorySpecifier` classes for cross-platform file operations
- **world.h, map.h** ΓÇö Game world/map data structures (not directly used; for context)
- **shell.h** ΓÇö Global functions (`pause_game`, `resume_game`, `show_cursor`, `hide_cursor`); `strFILENAMES` resource constants
- **interface.h** ΓÇö Game interface enums (`strFILENAMES` filenames, `strERRORS` error codes, `strPROMPTS` prompt strings)
- **game_wad.h** ΓÇö `save_game_file()`, `get_current_saved_game_name()` (defined elsewhere)
- **game_errors.h** ΓÇö Error type constants
- **shell_sdl.cpp (external)** ΓÇö `data_search_path` vector (defined elsewhere, used here)
- **STL vector** ΓÇö Dynamic array for search path iteration
