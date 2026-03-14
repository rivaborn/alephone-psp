# Source_Files/Files/game_wad.h

## File Purpose
Header declaring the game's save/load system and WAD (map data) management. Provides functions to persist game state to disk, manage save files and map checksums, and control pause/resume behavior. Interfaces with the engine's file handling subsystem.

## Core Responsibilities
- Save game state and export levels to disk
- Manage current saved game file specification
- Process and validate map WAD data structures
- Load maps and verify integrity via checksums
- Switch between map files
- Pause and resume game execution
- Retrieve save game file descriptors

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class | Encapsulates a file path/descriptor; used throughout for file operations |
| `wad_data` | struct | Game level/map data structure (definition elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### save_game_file
- Signature: `bool save_game_file(FileSpecifier& File)`
- Purpose: Serialize current game state to disk
- Inputs: File path/descriptor
- Outputs/Return: Boolean success status
- Side effects: I/O to disk; modifies file at path
- Calls: (not inferable from header)
- Notes: Used for manual save/checkpoint operations

### export_level
- Signature: `bool export_level(FileSpecifier& File)`
- Purpose: Save current level/map in exportable format
- Inputs: Destination file
- Outputs/Return: Boolean success
- Side effects: I/O to disk
- Calls: (not inferable)

### process_map_wad
- Signature: `bool process_map_wad(struct wad_data *wad, bool restoring_game, short version)`
- Purpose: Unpack and validate a map WAD, optionally restoring saved state
- Inputs: WAD data pointer; flag for game restoration; version number
- Outputs/Return: Boolean success
- Side effects: Modifies engine state (loads geometry, objects, etc.)
- Calls: (not inferable)
- Notes: Exposed for netgame resumption; critical path for map loading

### pause_game / resume_game
- Signature: `void pause_game(void)`, `void resume_game(void)`
- Purpose: Suspend/resume game simulation
- Side effects: Pause state affects frame update logic

### match_checksum_with_map
- Signature: `bool match_checksum_with_map(short vRefNum, long dirID, uint32 checksum, FileSpecifier& File)`
- Purpose: Verify map file integrity via checksum
- Inputs: Volume/directory reference; expected checksum; output file specifier
- Outputs/Return: Boolean match result; populates File on success
- Side effects: None (read-only validation)

### set_map_file / get_map_file
- Signature: `void set_map_file(FileSpecifier& File)`, `FileSpecifier& get_map_file(void)`
- Purpose: Switch current map file; retrieve active map file
- Notes: `get_map_file` returns reference to singleton current map

## Control Flow Notes
Part of the core game initialization and save/load pipeline. Called during menu navigation (save/load screens), level transitions, and pause/resume. `process_map_wad` is a critical engine entry point invoked when loading maps or resuming netgames.

## External Dependencies
- `FileSpecifier` (defined elsewhere; forward declared here)
- `wad_data` struct (definition not in this file)
- File I/O subsystem (implementation hidden)
