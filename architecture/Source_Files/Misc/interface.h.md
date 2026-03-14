# Source_Files/Misc/interface.h

## File Purpose
Master header declaring the public interface between major engine subsystems: game state management, UI/dialogs, shape/texture loading, input handling, recording/replay, and networking. Serves as the central coordination point for Marathon/Aleph One engine components.

## Core Responsibilities
- Define game state machine constants and controllers
- Declare game initialization, state transitions, and shutdown
- Expose shape/texture descriptor system and collection management
- Provide UI/dialog function declarations
- Declare input handling (keyboard, mouse, controller) functions
- Expose recording, replay, and networking APIs
- Define animation and shape rendering data structures
- Provide resource file loading coordination (maps, sounds, shapes, preferences)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `shape_information_data` | struct | Shape rendering flags (mirroring, obscuration), light intensity, and world bounding box |
| `shape_animation_data` | struct | Animation metadata: view count, frames/ticks, sounds, transfer mode, loop frame |
| Game State enum | enum | States: intro screens, main menu, game in progress, epilogue, quit dialogs |
| Game Controller enum | enum | Controllers: single player, network, demo, replay, replay from file |
| Error Codes enum | enum | Error IDs (memory, processor, file, network, serial validation errors) |
| Animation Types enum | enum | Animation frame counts: `_unanimated`, `_animated1`, `_animated2to8`, `_animated8`, etc. |
| Filename Indices enum | enum | 14 file type constants (shapes, sounds, preferences, maps, physics, music, images, theme) |
| Key Setup enum | enum | Keyboard preset: standard, left-handed, PowerBook, custom |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_game_state
- **Signature:** `void initialize_game_state(void);`
- **Purpose:** Initialize the game state machine and global game state variables.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Initializes game state; likely resets state to intro/menu.
- **Calls:** Not inferable from this file.
- **Notes:** Called during engine startup before main game loop.

### set_game_state / get_game_state
- **Signature:** `void set_game_state(short new_state);` / `short get_game_state(void);`
- **Purpose:** Transition game state machine and query current state.
- **Inputs:** `new_state` Γêê {intro, menu, chapter, prologue, epilogue, credits, game_in_progress, quit, etc.}
- **Outputs/Return:** Current state (int16)
- **Side effects:** May trigger UI/logic transitions.
- **Calls:** Not inferable.
- **Notes:** Central game state machine. One of: display screens, game in progress, or shutdown states.

### pause_game / resume_game
- **Signature:** `void pause_game(void);` / `void resume_game(void);`
- **Purpose:** Pause/unpause game simulation and UI updates.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Halts/resumes game tick processing and input handling.
- **Calls:** Not inferable.

### idle_game_state
- **Signature:** `bool idle_game_state(uint32 ticks);`
- **Purpose:** Per-frame game update; process tick actions, update world, render.
- **Inputs:** `ticks` elapsed since last call
- **Outputs/Return:** Boolean (success/continue flag)
- **Side effects:** Processes game simulation, input, and rendering.
- **Calls:** Not inferable.
- **Notes:** Core main loop entry point for game state updates.

### portable_process_screen_click
- **Signature:** `void portable_process_screen_click(short x, short y, bool cheatkeys_down);`
- **Purpose:** Handle mouse click input in game world or UI.
- **Inputs:** `x`, `y` screen coordinates; `cheatkeys_down` modifier
- **Outputs/Return:** None
- **Side effects:** May modify game state, UI, or player action.
- **Calls:** Not inferable.

### update_interface_display
- **Signature:** `void update_interface_display(void);`
- **Purpose:** Refresh HUD/UI display (health, ammo, map, etc.).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Redraws interface elements; affects video buffer.
- **Calls:** Not inferable.

### load_collections / mark_collection_for_loading
- **Signature:** `void load_collections(bool with_progress_bar, bool is_opengl);` / `#define mark_collection_for_loading(c) mark_collection((c), true)`
- **Purpose:** Queue and load shape/texture collections into memory.
- **Inputs:** `with_progress_bar` to show loading progress; `is_opengl` rendering mode flag; collection index
- **Outputs/Return:** None
- **Side effects:** Allocates memory, loads collection data from disk.
- **Calls:** Not inferable.
- **Notes:** Two-stage: queue collections with `mark_collection()`, then load all with `load_collections()`.

### get_shape_bitmap_and_shading_table
- **Signature:** `#define get_shape_bitmap_and_shading_table(shape, bitmap, shading_table, shading_mode) extended_get_shape_bitmap_and_shading_table(...)`
- **Purpose:** Retrieve bitmap and shading table for a shape descriptor.
- **Inputs:** `shape` descriptor, `shading_mode` (lighting mode)
- **Outputs/Return:** `bitmap` and `shading_table` pointers (output params)
- **Side effects:** None (read-only lookup).
- **Calls:** `extended_get_shape_bitmap_and_shading_table()`
- **Notes:** Macro wrapper; unpacks collection and shape from descriptor.

### process_action_flags
- **Signature:** `void process_action_flags(short player_identifier, const uint32 *action_flags, short count);`
- **Purpose:** Process recorded or replayed player input actions.
- **Inputs:** `player_identifier`, `action_flags` array, count of flags
- **Outputs/Return:** None
- **Side effects:** Applies input to player or recording buffer.
- **Calls:** Not inferable.
- **Notes:** Used by recording/replay system and network packet handling.

### network_gather / network_join
- **Signature:** `bool network_gather(bool inResumingGame);` / `int network_join(void);`
- **Purpose:** Host a networked game or join an existing one.
- **Inputs:** `inResumingGame` for host; none for join
- **Outputs/Return:** Bool (success) or int (join result: failed/new/resume)
- **Side effects:** Establishes network connection, loads game state, initializes netplay.
- **Calls:** Not inferable.
- **Notes:** Gateway to multiplayer gameplay.

### load_game / save_game / revert_game
- **Signature:** `bool load_game(bool use_last_load);` / `bool save_game(void);` / `bool revert_game(void);`
- **Purpose:** Persist/restore game state to/from disk.
- **Inputs:** `use_last_load` to skip dialog
- **Outputs/Return:** Bool (success)
- **Side effects:** File I/O; modifies game world and player state.
- **Calls:** Not inferable.
- **Notes:** `revert_game()` restores last checkpoint; requires prior setup via `setup_revert_game_info()`.

### handle_preferences_dialog / handle_load_game / handle_save_game
- **Signature:** `bool handle_preferences_dialog(void);` / `void handle_load_game(void);` / `void handle_save_game(void);`
- **Purpose:** Display and handle UI dialogs for game preferences, load, and save.
- **Inputs:** None
- **Outputs/Return:** Bool or void
- **Side effects:** Modifies preferences; loads/saves game state; blocks until dialog closed.
- **Calls:** Not inferable.
- **Notes:** Blocking dialogs; typically called from main menu or pause screen.

### Infravision_GetParser / ControlPanels_GetParser
- **Signature:** `XML_ElementParser *Infravision_GetParser();` / `XML_ElementParser *ControlPanels_GetParser();`
- **Purpose:** Retrieve XML parsers for infravision and control-panel configuration.
- **Inputs:** None
- **Outputs/Return:** Pointer to `XML_ElementParser` instance (defined elsewhere)
- **Side effects:** None (factory function).
- **Calls:** Not inferable.
- **Notes:** Allows external modules to parse XML definition files.

## Notes
- Trivial helper declarations (e.g., `get_mouse_position()`, `hide_cursor()`, `show_cursor()`, `scroll_inventory()`) omitted for brevity; these are platform-specific UI stubs.
- Recording/replay functions (`rewind_recording()`, `stop_recording()`, `stop_replay()`, `check_recording_replaying()`, `increment_replay_speed()`, etc.) provide playback control for demos and file-based replays.
- Behavior modifier functions (`dont_switch_to_new_weapon()`, `dont_auto_recenter()`, `standardize_player_behavior_modifiers()`) allow network clients to enforce consistent behavior across players.
- File specifiers (`get_default_map_spec()`, `get_default_physics_spec()`, etc.) abstract resource location; used to support external file searching on macOS.

## Control Flow Notes
**Initialization ΓåÆ Frame Loop ΓåÆ Shutdown:**
1. **Init:** `initialize_game_state()`, `init_physics_wad_data()`, `load_collections()` with progress bar
2. **Main Loop:** `idle_game_state(ticks)` called per frame; handles input, updates world, schedules rendering
3. **State Transitions:** Game state machine via `set_game_state()`; e.g., intro ΓåÆ menu ΓåÆ game in progress ΓåÆ epilogue ΓåÆ quit
4. **Dialogs:** Blocking calls (preferences, load, save) pause main loop during user interaction
5. **Recording/Replay:** `process_action_flags()` replaces direct input; integrated into frame update
6. **Networking:** `network_gather()/network_join()` establish session; then normal `idle_game_state()` loop processes networked input
7. **Shutdown:** `unload_all_collections()`, `exit_networking()` on quit

## External Dependencies
- **Includes:** `XML_ElementParser.h` (XML parsing utilities)
- **Forward decls:** `FileSpecifier` class (file specification from elsewhere)
- **Used types from `shape_descriptors.h`:** `shape_descriptor` typedef, collection enum, macros (`GET_DESCRIPTOR_COLLECTION`, `BUILD_DESCRIPTOR`)
- **Defined elsewhere, used here:** `bitmap_definition` struct, `rgb_color_value` struct, `game_data`, `entry_point`, `player_start_data`, `low_level_shape_definition`, `_fixed` type
