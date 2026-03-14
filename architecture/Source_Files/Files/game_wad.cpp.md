# Source_Files/Files/game_wad.cpp

## File Purpose
Manages loading and saving of game levels/maps from WAD (data archive) files. Handles game state serialization, level initialization, network map transfer, and coordinates the complex initialization sequence when entering a map (geometry, objects, monsters, platforms, etc.).

## Core Responsibilities
- Load map data from WAD files and initialize game world (geometry, objects, monsters, platforms, lights)
- Save game state to WAD files (both savegames and level exports)
- Manage map file selection and validation (`set_map_file()`, `use_map_file()`)
- Initiate new games and handle level transitions (`new_game()`, `goto_level()`)
- Support network map data transfer for multiplayer games
- Provide map entry points (spawn locations) for players
- Pack/unpack game data structures to/from binary streams for persistence
- Handle game state recovery and revert functionality

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `revert_game_info` | struct | Stores game state for revert-game functionality (game data, player start info, entry point, saved game file) |
| `save_game_data` | struct | Metadata describing what data to save/export (tag, unit size, level-loading flag) |
| `game_data`, `dynamic_data`, `static_data` | struct | Game state; see map.h |
| `wad_data`, `wad_header` | struct | WAD file format structures; see wad.h |
| Multiple `*_data` structs | struct | Map geometry, objects, platforms, monsters, effects, projectiles, lights, etc.; see map.h and related headers |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MapFileSpec` | FileSpecifier | global | Currently loaded map file |
| `file_is_set` | bool | static | Whether a valid map file has been set |
| `PhysicsModelLoadedEarlier` | bool | static | Tracks if physics model was loaded from previous level (avoid redundant loading) |
| `revert_game_data` | revert_game_info | static | Stores game info for manual game revert (resume after abort) |
| `static_platforms` | vector\<static_platform_data\> | static | Cached platform static data for level export |

## Key Functions / Methods

### set_map_file
- **Signature:** `void set_map_file(FileSpecifier& File)`
- **Purpose:** Set the active map file and initialize related resources (scenario images, level scripts).
- **Inputs:** FileSpecifier reference to map file.
- **Outputs/Return:** None.
- **Side effects:** Updates global `MapFileSpec`, calls `RunRestorationScript()`, loads level scripts, clears game errors.
- **Calls:** `RunRestorationScript()`, `set_scenario_images_file()`, `LoadLevelScripts()`, `clear_game_error()`.
- **Notes:** Runs parameter restoration script before switching files (e.g., texture reloads).

### load_level_from_map
- **Signature:** `bool load_level_from_map(short level_index)`
- **Purpose:** Load and process a level from the currently set map file.
- **Inputs:** Level index (NONE = restore saved game).
- **Outputs/Return:** true on success, false on error.
- **Side effects:** Reads WAD from file, unpacks all level data, processes map, updates global game state.
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `process_map_wad()`, `free_wad()`, `close_wad_file()`, `error_pending()`.
- **Notes:** Handles both new-game loads and savegame restores; error state set in game_errors.c.

### new_game
- **Signature:** `bool new_game(short number_of_players, bool network, struct game_data *game_information, struct player_start_data *player_start_information, struct entry_point *entry_point)`
- **Purpose:** Initialize a new game: set up players, load level, initialize world state.
- **Inputs:** Player count, network flag, game info, player starts, entry point.
- **Outputs/Return:** true on success.
- **Side effects:** Initializes players, calls `goto_level()`, sets up action queues, resets motion sensor, initializes chase cam, stores revert game info.
- **Calls:** `initialize_map_for_new_game()`, `goto_level()`, `new_player()`, `entering_map()`, `reset_action_queues()`, `reset_motion_sensor()`, `ChaseCam_Initialize()`, `setup_revert_game_info()`.
- **Notes:** Order matters: map initialized before players; players created after level load (monsters depend on proper world state).

### goto_level
- **Signature:** `bool goto_level(struct entry_point *entry, bool new_game)`
- **Purpose:** Transition to a level; load map, initialize objects, run level script.
- **Inputs:** Entry point (level number, name), new-game flag.
- **Outputs/Return:** true on success.
- **Side effects:** Calls `load_level_from_map()`, runs level scripts, initializes monsters, processes entry point.
- **Calls:** `load_level_from_map()`, `RunLevelScript()`, `entering_map()`.
- **Notes:** Critical for level transitions; must run script before collection loading.

### process_map_wad
- **Signature:** `bool process_map_wad(struct wad_data *wad, bool restoring_game, short version)`
- **Purpose:** Core unpacking: extract and process all geometry, objects, lights, platforms from WAD.
- **Inputs:** WAD data, restore flag, version.
- **Outputs/Return:** true on success.
- **Side effects:** Allocates map memory, unpacks all structures into global arrays, calls completion functions.
- **Calls:** `allocate_map_structure_for_map()`, `load_*()` functions, `complete_restoring_level()`, `complete_loading_level()`.
- **Notes:** Handles both old (Marathon 1) and new WAD formats; version-dependent logic.

### build_save_game_wad / build_export_wad
- **Signature:** `static struct wad_data *build_save_game_wad(struct wad_header *header, long *length)`
- **Purpose:** Serialize entire game state (or map for export) into a WAD structure.
- **Inputs:** WAD header, output length pointer.
- **Outputs/Return:** Pointer to newly constructed wad_data.
- **Side effects:** Allocates memory for packed data, recalculates counts, appends data chunks to WAD.
- **Calls:** `create_empty_wad()`, `recalculate_map_counts()`, `tag_to_global_array_and_size()`, `append_data_to_wad()`, `calculate_wad_length()`.
- **Notes:** export_wad resets platform/polygon states; save_wad preserves runtime state. Both delete packed arrays after use.

### get_indexed_entry_point / get_entry_points
- **Signature:** `bool get_indexed_entry_point(struct entry_point *entry_point, short *index, int32 type)` / `bool get_entry_points(vector<entry_point> &vec, int32 type)`
- **Purpose:** Query available map entry points (spawn locations) matching a type (single-player, multiplayer, etc.).
- **Inputs:** Output entry_point pointer, start index, entry point type flags.
- **Outputs/Return:** true if found; output structure filled with level number and name.
- **Side effects:** Opens map file, reads header and directory data.
- **Calls:** `open_wad_file_for_reading()`, `read_wad_header()`, `read_directory_data()`, `unpack_directory_data()`, `extract_type_from_wad()`, `unpack_static_data()`.
- **Notes:** Handles both old and new WAD directory formats.

### get_player_starting_location_and_facing
- **Signature:** `short get_player_starting_location_and_facing(short team, short index, struct object_location *location)`
- **Purpose:** Retrieve player spawn location and facing angle.
- **Inputs:** Team index (NONE for any), location index, output location pointer.
- **Outputs/Return:** Count of available spawns; fills location if pointer non-null.
- **Side effects:** Scans saved_objects array.
- **Calls:** (Direct array access only.)
- **Notes:** location==nullptr returns count; team==NONE matches any team.

### tag_to_global_array_and_size
- **Signature:** `static uint8 *tag_to_global_array_and_size(uint32 tag, size_t *size)`
- **Purpose:** Serialize a single data category (endpoint, line, monster, etc.) from global arrays.
- **Inputs:** WAD tag identifier, output size pointer.
- **Outputs/Return:** Pointer to newly allocated packed byte array (or NULL if size==0).
- **Side effects:** Allocates memory; packs data via pack_*() functions.
- **Calls:** Large switch on tag; calls pack_endpoint_data(), pack_line_data(), pack_player_data(), etc.
- **Notes:** Caller must delete returned array. Size set to 0 returns NULL; no allocation.

### complete_loading_level / complete_restoring_level
- **Signature:** `void complete_loading_level(short *_map_indexes, size_t map_index_count, uint8 *_platform_data, size_t platform_data_count, uint8 *actual_platform_data, size_t actual_platform_data_count, short version)` / `static void complete_restoring_level(struct wad_data *wad)`
- **Purpose:** Final initialization after map data unpacked: redundant data, platforms, scenery, lights (Marathon 1 compatibility).
- **Inputs:** Map indexes, platform data, version.
- **Outputs/Return:** None.
- **Side effects:** Loads map indexes, adds platforms and scenery, recalculates light sources.
- **Calls:** `load_redundant_map_data()`, `scan_and_add_platforms()`, `scan_and_add_scenery()`, `guess_side_lightsource_indexes()`, `reset_action_queues()`.
- **Notes:** Must run after geometry unpacked but before entering map.

### Network Functions (process_net_map_data, get_map_for_net_transfer)
- **Signature:** `bool process_net_map_data(void *data)` / `void *get_map_for_net_transfer(struct entry_point *entry)`
- **Purpose:** Send/receive map data over network in multiplayer games.
- **Inputs:** Flat data buffer (net) or entry point (transfer).
- **Outputs/Return:** Boolean success or void* to flat data.
- **Side effects:** Inflates/deflates WAD data, processes via `process_map_wad()`.
- **Calls:** `inflate_flat_data()`, `process_map_wad()`, `free_wad()`, `get_flat_data()`.
- **Notes:** Flat data is compressed; WAD is expanded for processing.

## Control Flow Notes
This file orchestrates the level loading pipeline:

1. **Initialization:** `set_map_file()` ΓåÆ `new_game()` ΓåÆ `goto_level()` ΓåÆ `load_level_from_map()` ΓåÆ `process_map_wad()` ΓåÆ `complete_loading_level()` ΓåÆ `entering_map()`
2. **Level Transition:** `goto_level()` repeats the load sequence for a new level.
3. **Saving:** `build_save_game_wad()` serializes current state for savegame; `build_export_wad()` exports map without runtime changes.
4. **Restoration:** `load_level_from_map(NONE)` restores a savegame; `complete_restoring_level()` runs cleanup.
5. **Network:** `process_net_map_data()` inflates and processes incoming map data; `get_map_for_net_transfer()` deflates for sending.

Version handling is critical: old Marathon 1 maps handled via version checks (e.g., recalculating map indexes).

## External Dependencies
- **Notable includes:** map.h, monsters.h, network.h, projectiles.h, effects.h, player.h, platforms.h, wad.h, FileHandler.h, Packing.h, computer_interface.h, XML_LevelScript.h, ChaseCam.h, Music.h, SoundManager.h.
- **Defined elsewhere (partial list):**
  - Core structures: `wad_data`, `wad_header`, `game_data`, `dynamic_world`, `static_world`, `entry_point`, `map_object`, `static_data`, etc. (map.h, wad.h)
  - Globals: `objects`, `map_endpoints`, `map_lines`, `map_sides`, `map_polygons`, `monsters`, `effects`, `projectiles`, `platforms`, `saved_objects`, `map_indexes`, `players`, `dynamic_world`, `automap_lines`, `automap_polygons`
  - Functions: `open_wad_file_for_reading()`, `read_wad_header()`, `extract_type_from_wad()`, `append_data_to_wad()`, `create_empty_wad()`, `inflate_flat_data()`, `get_flat_data()`, `free_wad()`, many pack/unpack functions (Packing.h)
  - Game callbacks: `entering_map()`, `leaving_map()`, `goto_level()`, `RunLevelScript()`, `RunRestorationScript()`, `initialize_map_for_new_game()`, `new_player()`, `new_monster()`, `entering_map()`, `reset_motion_sensor()`, `ChaseCam_Initialize()`
