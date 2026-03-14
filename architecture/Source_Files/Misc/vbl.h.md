# Source_Files/Misc/vbl.h

## File Purpose
Defines the interface for replay recording and playback functionality in the game engine. Handles setup, streaming, and playback of recorded game sessions, including keyboard input capture, heartbeat synchronization, and XML-based keyboard configuration.

## Core Responsibilities
- Setting up replay playback from files or random resources
- Recording and retrieving game session header data (players, level, checksum, version)
- Managing input controller state and heartbeat synchronization
- Keyboard initialization and keymap parsing
- File operations for replay discovery and movement
- XML-based keyboard configuration parsing
- Debug replay stream logging (conditional compilation)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `recorded_flag` | struct | Debug-only: tracks action flags with associated player index |

**Referenced structures** (defined elsewhere):
- `player_start_data` ΓÇô Player spawn/start position data
- `game_data` ΓÇô Game state and configuration
- `FileSpecifier` ΓÇô Cross-platform file specification (from FileHandler.h)
- `XML_ElementParser` ΓÇô XML parsing interface for keyboard config

## Global / File-Static State
None.

## Key Functions / Methods

### `setup_for_replay_from_file`
- Signature: `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum);`
- Purpose: Initialize replay playback from a specified file
- Inputs: File reference, map checksum for validation
- Outputs/Return: `true` if setup succeeded; `false` otherwise
- Calls: (depends on VBL.C implementation)

### `start_recording`
- Signature: `void start_recording(void);`
- Purpose: Begin recording the current game session
- Side effects: Initializes recording state

### `find_replay_to_use`
- Signature: `bool find_replay_to_use(bool ask_user, FileSpecifier& File);`
- Purpose: Locate an available replay file, optionally prompting user
- Inputs: Flag to enable user dialog; output file reference
- Outputs/Return: `true` if replay found; `false` otherwise

### `set_recording_header_data` / `get_recording_header_data`
- Signature: `void set_recording_header_data(short num_players, short level_num, uint32 map_checksum, short version, struct player_start_data *starts, struct game_data *game_info);`
- Purpose: Store/retrieve session metadata (players, level, checksum, version, spawn points, game state)
- Notes: Paired operations for persistence

### `input_controller`
- Signature: `bool input_controller(void);`
- Purpose: Poll and process input state (record or playback)
- Outputs/Return: `true` if input successfully processed

### `increment_heartbeat_count`
- Signature: `void increment_heartbeat_count(int value = 1);`
- Purpose: Advance frame/tick counter for synchronization (default +1)
- Side effects: Updates internal heartbeat state

### `initialize_keyboard_controller`
- Signature: `void initialize_keyboard_controller(void);`
- Purpose: Set up keyboard input handling and binding system

### `parse_keymap`
- Signature: `uint32 parse_keymap(void);`
- Purpose: Read and decode keyboard mapping from resource or file
- Outputs/Return: Parsed keymap as 32-bit value

### `Keyboard_GetParser`
- Signature: `XML_ElementParser *Keyboard_GetParser();`
- Purpose: Return parser object for XML-based keyboard configuration
- Notes: Enables XML-driven input configuration

**Trivial helpers** (omitted from full entries):
- `get_recording_filedesc` ΓÇô Locate recording file descriptor
- `move_replay` ΓÇô Relocate or rename replay file

## Control Flow Notes
This module integrates into the game's main update/frame loop:
- **Init**: `initialize_keyboard_controller()` runs at startup
- **Frame**: `input_controller()` polled each frame; `increment_heartbeat_count()` maintains sync
- **Recording**: `start_recording()` + `set_recording_header_data()` capture session data
- **Playback**: `setup_for_replay_from_file()` + subsequent `input_controller()` calls replay input

## External Dependencies
- **FileHandler.h** ΓÇô `FileSpecifier` class for platform-agnostic file I/O
- **tags.h** ΓÇô Typecodes for file types (included via FileHandler.h)
- Undefined references: `player_start_data`, `game_data`, `XML_ElementParser` structures

---

**Note**: Debug replay streaming (`open_stream_file`, `write_flags`, `debug_stream_of_flags`, `close_stream_file`) is only compiled when `DEBUG_REPLAY` is defined; see conditional block for details.
