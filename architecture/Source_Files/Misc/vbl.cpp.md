# Source_Files/Misc/vbl.cpp

## File Purpose

Manages keyboard input polling, action flag processing, and game replay recording/playback. Originally the "VBL controller" for frame-synchronous player movement on classic Mac, it now acts as the input subsystem that collects keypresses, converts them to action flags, and distributes them to the game engine and recording system. Also handles replaying recorded games.

## Core Responsibilities

- Poll keyboard/mouse input and convert to standardized action flag bit patterns
- Install and maintain a periodic timer task that drives input polling at ~30 Hz
- Record and playback game inputs to/from film files for demo and replay functionality
- Manage keyboard key mappings and support multiple predefined key layouts (standard, left-handed, PowerBook)
- Handle special input behaviors (double-click detection, latched keys, run/walk and swim/sink modifiers)
- Maintain circular action flag queues per player for recorded input
- Parse and serialize replay file headers containing game metadata
- Provide XML configuration interface for keyboard setup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `key_definition` | struct | Maps a physical key (SDLKey or raw code) to an action flag |
| `special_flag_data` | struct | Metadata for double-click and latched-key behavior (persistence counter, flag bits) |
| `replay_private_data` | struct | Opaque replay state: queues, file handles, headers, cache buffers for reading/writing film files |
| `recording_header` | struct | Serialized metadata: player count, level, map checksum, game settings, player start positions |
| `ActionQueue` | struct | Circular buffer (read/write indices, fixed-size buffer) for one player's recorded action flags |
| `XML_KeyParser` | class | XML element parser for individual key binding entries (index + platform-specific keycode) |
| `XML_KeyboardParser` | class | XML element parser for keyboard setup selection (which predefined layout) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `heartbeat_count` | long | static | Frame counter incremented by input task; used to detect dropped frames |
| `input_task_active` | bool | static | Whether input polling is currently enabled |
| `input_task` | timer_task_proc | static | Installed timer task handle for cleanup on exit |
| `FilmFileSpec` | FileSpecifier | static | File path for current replay recording/playback |
| `FilmFile` | OpenedFile | static | Open file handle for replay I/O |
| `replay` | replay_private_data | static | All replay state (queues, headers, offsets, cache, file handles) |
| `tm_func` | timer_func | static | Pointer to installed timer callback (SDL port only) |
| `tm_period` | uint32 | static | Millisecond period between timer task calls (SDL port) |
| `tm_last`, `tm_accum` | uint32 | static | Accumulator and last-call timestamp for SDL timer (SDL port) |
| `current_key_definitions` | key_definition[] | extern file | Current active key mapping (not static; linked elsewhere) |
| `XML_KeySet` | key_definition* | static | Pointer to the key set being parsed by XML parser |

## Key Functions / Methods

### initialize_keyboard_controller
- **Signature:** `void initialize_keyboard_controller(void)`
- **Purpose:** One-time initialization of the input system at game startup
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates replay queues (`MAXIMUM_NUMBER_OF_PLAYERS` ActionQueue objects), installs timer task, registers exit handler, loads default key mappings, enters mouse mode
- **Calls:** `install_timer_task()`, `atexit(remove_input_controller)`, `set_keys_to_match_preferences()`, `enter_mouse()`
- **Notes:** Asserts key definition array sizes match; initializes `heartbeat_count` to 0

### set_keyboard_controller_status
- **Signature:** `void set_keyboard_controller_status(bool active)`
- **Purpose:** Enable/disable input polling (pauses game if false)
- **Inputs:** `active` ΓÇô whether input should be processed
- **Outputs/Return:** None
- **Side effects:** Sets `input_task_active`; on Mac calls `Start_ISp()` / `Stop_ISp()` (Input Sprocket); on SDL calls `enter_mouse()` / `exit_mouse()` with device type
- **Calls:** `Start_ISp()`, `Stop_ISp()`, `enter_mouse()`, `exit_mouse()`, `mouse_idle()`
- **Notes:** Input Sprocket is Mac-only legacy; SDL conditionals handle cross-platform mouse control

### input_controller
- **Signature:** `bool input_controller(void)`
- **Purpose:** Main timer task callback; drives all input polling and action flag generation each frame
- **Inputs:** None (called by timer system)
- **Outputs/Return:** `true` to reschedule the task
- **Side effects:** Increments `heartbeat_count`, pulls or generates action flags, calls `process_action_flags()` or `pull_flags_from_recording()`
- **Calls:** `process_action_flags()`, `pull_flags_from_recording()`, `parse_keymap()`, `set_game_state()`
- **Notes:** Checks phase/sync with `dynamic_world->tick_count` to prevent dropped or repeated frames; handles replay speed control and end-of-file detection

### parse_keymap
- **Signature:** `uint32 parse_keymap(void)`
- **Purpose:** Scans current keyboard state (SDL keymap or Console input if chat active) and returns composite action flags
- **Inputs:** None (reads global state)
- **Outputs/Return:** 32-bit action flag word combining all pressed inputs
- **Side effects:** Updates `special_flags[]` persistence counters for double-click and latched-key behavior; calls `test_mouse()` if mouse input device active; applies run/walk and swim/sink interchange based on player state and preferences; handles PSP-specific gamepad input
- **Calls:** `SDL_GetKeyState()`, `Console::instance()->input_active()`, `test_mouse()`, `mouse_buttons_become_keypresses()`, `mask_in_absolute_positioning_information()`, `build_terminal_action_flags()`, `player_in_terminal_mode()`
- **Notes:** Complex state machine for special flags; PSP code path is separate and hardcoded; applies input modifiers from preferences

### process_action_flags
- **Signature:** `void process_action_flags(short player_identifier, const uint32 *action_flags, short count)`
- **Purpose:** Record action flags if recording, then enqueue them for network/gameplay
- **Inputs:** `player_identifier` ΓÇô which player, `action_flags` ΓÇô array of flags, `count` ΓÇô number of frames worth
- **Outputs/Return:** None
- **Side effects:** If recording, calls `record_action_flags()`; always calls `GetRealActionQueues()->enqueueActionFlags()`
- **Calls:** `record_action_flags()`, `GetRealActionQueues()->enqueueActionFlags()`
- **Notes:** Acts as a dispatch point between recording and gameplay systems

### start_recording
- **Signature:** `void start_recording(void)`
- **Purpose:** Begin a new replay recording session
- **Inputs:** None (uses global `replay` state and `FilmFileSpec`)
- **Outputs/Return:** None
- **Side effects:** Deletes any existing film file, creates new file, packs recording header to file, sets `replay.game_is_being_recorded = true`
- **Calls:** `FilmFileSpec.Delete()`, `FilmFileSpec.Create()`, `FilmFileSpec.Open()`, `pack_recording_header()`, `FilmFile.Write()`
- **Notes:** Must have called `set_recording_header_data()` first to populate the header; asserts `!replay.valid` on entry

### stop_recording
- **Signature:** `void stop_recording(void)`
- **Purpose:** End replay recording, flush final chunks, rewrite header with correct file length
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Flushes each player's queue chunk to disk, seeks to file start and rewrites header with updated length, closes file, clears `replay.valid`
- **Calls:** `save_recording_queue_chunk()`, `FilmFile.SetPosition()`, `pack_recording_header()`, `FilmFile.Write()`, `FilmFile.GetLength()`, `FilmFile.Close()`
- **Notes:** Asserts final file length matches header; uses size-prefixed serialization for replay data

### setup_for_replay_from_file
- **Signature:** `bool setup_for_replay_from_file(FileSpecifier& File, uint32 map_checksum)`
- **Purpose:** Load a replay file and prepare for playback
- **Inputs:** `File` ΓÇô path to replay file, `map_checksum` ΓÇô (unused) expected map identifier
- **Outputs/Return:** `true` if successful, `false` if map not found
- **Side effects:** Opens film file, unpacks header, allocates disk read cache, validates map file, initializes replay speed to 1x, sets `replay.game_is_being_replayed = true`
- **Calls:** `FilmFileSpec.Open()`, `FilmFile.Read()`, `unpack_recording_header()`, `use_map_file()`, `alert_user()`
- **Notes:** Allocates either `resource_data` (if embedded) or `fsread_buffer` for file I/O caching

### pull_flags_from_recording
- **Signature:** `static bool pull_flags_from_recording(short count)`
- **Purpose:** Load `count` action flags from each player's replay queue and enqueue for gameplay
- **Inputs:** `count` ΓÇô number of frames of input to pull per player
- **Outputs/Return:** `true` if queues had data, `false` if any queue was empty (start of game)
- **Side effects:** Reads from each player's queue buffer in `replay`, increments read indices, calls `GetRealActionQueues()->enqueueActionFlags()` per flag
- **Calls:** `get_recording_queue_size()`, `GetRealActionQueues()->enqueueActionFlags()`
- **Notes:** Asserts queue read/write indices don't collide; early exit if any queue is empty (protective measure at level start)

### save_recording_queue_chunk
- **Signature:** `void save_recording_queue_chunk(short player_index)`
- **Purpose:** Serialize one player's recorded action flags to disk using run-length encoding
- **Inputs:** `player_index` ΓÇô which player's queue to flush
- **Outputs/Return:** None
- **Side effects:** Reads up to `RECORD_CHUNK_SIZE` flags from queue, encodes runs of identical flags, writes to `FilmFile`, updates `replay.header.length`
- **Calls:** `get_player_recording_queue()`, `get_recording_queue_size()`, `ValueToStream()`, `FilmFile.Write()`
- **Notes:** Uses static buffer allocation for efficiency; run-length encoding compresses sequences of the same action flag

### Keyboard_GetParser
- **Signature:** `XML_ElementParser *Keyboard_GetParser(void)`
- **Purpose:** Export XML parser for keyboard configuration from maps or scripts
- **Inputs:** None
- **Outputs/Return:** Pointer to `KeyboardParser` with `KeyParser` added as child
- **Side effects:** None (parser object is static)
- **Calls:** `KeyboardParser.AddChild()`
- **Notes:** Allows external code to parse keyboard setup from XML without exposing internal parser classes

### install_timer_task (SDL only)
- **Signature:** `timer_task_proc install_timer_task(short tasks_per_second, timer_func func)`
- **Purpose:** Register a function to be called periodically (SDL port only; Mac would use OS interrupt)
- **Inputs:** `tasks_per_second` ΓÇô frequency (30 for ~30 Hz), `func` ΓÇô callback
- **Outputs/Return:** Opaque handle for later removal
- **Side effects:** Stores period, function pointer, initializes accumulator; gets current time via `SDL_GetTicks()`
- **Calls:** `SDL_GetTicks()`
- **Notes:** Single-task limitation by design (adequate for input); real timing done in `execute_timer_tasks()`

### execute_timer_tasks (SDL only)
- **Signature:** `void execute_timer_tasks(uint32 time)`
- **Purpose:** Called from main loop to service the installed timer task based on elapsed time
- **Inputs:** `time` ΓÇô current tick count from `SDL_GetTicks()`
- **Outputs/Return:** None
- **Side effects:** Accumulates time delta, calls registered timer function when period elapses, calls `mouse_idle()` once per accumulated period
- **Calls:** `tm_func()`, `mouse_idle()`
- **Notes:** Decouples input polling from frame rate by accumulating time independently

## Control Flow Notes

**Initialization ΓåÆ Runtime ΓåÆ Shutdown:**
1. `initialize_keyboard_controller()` is called at game startup; allocates queues and installs timer task
2. Main game loop calls `execute_timer_tasks(SDL_GetTicks())` each frame (SDL port)
3. When enough time accumulates, `input_controller()` callback fires:
   - If actively playing: `parse_keymap()` ΓåÆ `process_action_flags()` ΓåÆ recording + enqueue
   - If replaying: `pull_flags_from_recording()` ΓåÆ enqueue
   - If in network: handled elsewhere (comment indicates this)
4. Periodically, `check_recording_replaying()` is called to manage chunk I/O (flush on record, load on replay)
5. On exit, `remove_input_controller()` (registered via `atexit()`) flushes replay and closes files

**Recording flow:** Start (header written) ΓåÆ per-frame (action flags accumulated in circular queue) ΓåÆ periodic chunk flush to disk ΓåÆ stop (rewrite header with final length)

**Replay flow:** Load header ΓåÆ fill queues from chunks ΓåÆ per-frame (pull from queues and enqueue) ΓåÆ detect EOF and end replay

## External Dependencies

- **Includes / Imports:**
  - `cseries.h` ΓÇô cross-platform utilities (macros, types, memory)
  - `map.h` ΓÇô game world data (world coordinates, object definitions, game state)
  - `interface.h` ΓÇô game state machine and UI
  - `shell.h` ΓÇô window/screen constants
  - `preferences.h` ΓÇô player and input preferences
  - `mouse.h` ΓÇô mouse input polling
  - `player.h` ΓÇô player data structures
  - `key_definitions.h` ΓÇô static key mapping tables
  - `tags.h` ΓÇô not inferable from usage
  - `vbl.h` ΓÇô own public interface
  - `ISp_Support.h` ΓÇô Input Sprocket (Mac legacy)
  - `FileHandler.h` ΓÇô file I/O abstraction
  - `Packing.h` ΓÇô byte serialization utilities
  - `ActionQueues.h` ΓÇô action flag queue management
  - `computer_interface.h` ΓÇô probably AI or console
  - `Console.h` ΓÇô in-game chat console
  - `pspctrl.h`, `psp_common.h` ΓÇô PSP gamepad support (conditional)
  - `SDL.h` ΓÇô cross-platform input/time (conditional on SDL)

- **External symbols used (defined elsewhere):**
  - `dynamic_world` ΓÇô global game state (tick_count, player_count, etc.)
  - `local_player_index`, `local_player`, `current_player` ΓÇô player accessors
  - `input_preferences` ΓÇô user input settings
  - `graphics_preferences` ΓÇô screen/video settings
  - `GetRealActionQueues()` ΓÇô singleton for action flag distribution
  - `game_is_networked` ΓÇô network game flag
  - `static_world` ΓÇô map metadata
  - `set_game_state()`, `get_game_state()` ΓÇô game state machine
  - `player_in_terminal_mode()` ΓÇô player UI state query
  - `build_terminal_action_flags()` ΓÇô terminal input handling
  - `enter_mouse()`, `exit_mouse()`, `test_mouse()`, `mouse_idle()` ΓÇô mouse subsystem
  - `test_mouse()` ΓÇô mouse polling
  - `mask_in_absolute_positioning_information()` ΓÇô physics helper
  - `use_map_file()` ΓÇô map loading
  - `alert_user()` ΓÇô user dialogs
  - `Start_ISp()`, `Stop_ISp()` ΓÇô Input Sprocket control (Mac)
