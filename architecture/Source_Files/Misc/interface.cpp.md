# Source_Files/Misc/interface.cpp

## File Purpose

Central state machine and controller for the game's high-level flowΓÇömanages transitions between intro screens, main menu, gameplay, epilogue, and credits. Coordinates screen rendering, game initialization/cleanup, network setup, and player start configuration for single-player, multiplayer, and replay modes.

## Core Responsibilities

- **Game state machine**: Tracks and transitions between intro, menu, gameplay, epilogue, and quit states.
- **Screen sequencing**: Displays and advances through timed intro/credit screens and chapter headings.
- **Screen/interface fading**: Implements cinematic fade-in/fade-out effects with color table manipulation.
- **Game startup**: Initializes new games and resumes saved games for single and network play.
- **Player start synchronization**: Matches saved/live players to start positions and constructs player roster for games.
- **Network game orchestration**: Gathers players, handles network resume, installs/removes microphone for netplay.
- **Menu interaction**: Processes screen clicks on interface buttons and directs menu commands.
- **Game lifecycle**: Handles pause, finish, cleanup, and error recovery for failed game loads.

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `game_state` | struct | Tracks current state, phase counter, screen index, flags for background task suppression and microphone support |
| `screen_data` | struct | Defines screen sequence: base resource ID, count, duration in ticks |
| `player_start_data` | struct (from map.h) | Player team, color, name, identifier with auto-recenter/weapon-switch flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `game_state` | `struct game_state` | static | Primary state machine tracker (state, phase, current_screen, flags) |
| `DraggedReplayFile` | `FileSpecifier` | static | File handle for user-dragged replay file |
| `interface_fade_in_progress` | `bool` | static | Whether a fade animation is currently active |
| `interface_fade_type` | `short` | static | Fade direction/style (cinematic fade in/out) |
| `current_picture_clut` | `struct color_table*` | static | Active color palette for screen display; NULL when not fading |
| `current_picture_clut_depth` | `short` | static | Bit depth of current color table |
| `animated_color_table` | `struct color_table*` | static | Working color table during fade animation |
| `display_screens[]` | `struct screen_data[]` | static const | Lookup table defining all display sequences |

## Key Functions / Methods

### initialize_game_state
- Signature: `void initialize_game_state(void)`
- Purpose: Boot the game to intro screens; set initial state and suppress background tasks during load.
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies `game_state` (sets to intro), hides menus, calls `display_introduction()`
- Calls: `display_introduction()`, `toggle_menus()`

### idle_game_state
- Signature: `bool idle_game_state(uint32 ticks)`
- Purpose: Main game loop updateΓÇöadvances state machine, updates screen displays, processes fades, and ticks the world forward.
- Inputs: `ticks` (elapsed time since last call)
- Outputs/Return: `true` if world needs rendering
- Side effects: Advances `game_state.phase`, renders screens, updates player/world, queues network sync
- Calls: `update_interface_fades()`, `update_world()`, `NetSync()`, `next_game_screen()`, extensive conditional dispatch

### set_game_state
- Signature: `void set_game_state(short new_state)`
- Purpose: State-machine transition handler; enforces valid transitions and defers network-sensitive changes to idle time.
- Inputs: `new_state` (target game state)
- Outputs/Return: None
- Side effects: Updates `game_state.state` and `game_state.phase`; may call `finish_game()` or transition to intermediate states
- Calls: `finish_game()`, `display_quit_screens()`
- Notes: Only processes transitions from `_game_in_progress` directly; others go straight to new state. Defers `_switch_demo` and `_revert_game` to idle.

### next_game_screen
- Signature: `void next_game_screen(void)`
- Purpose: Advance to the next screen in a timed sequence (intro/credits); start fade and load new picture if it exists.
- Inputs: None
- Outputs/Return: None
- Side effects: Increments `game_state.current_screen`; may call `display_main_menu()`, `interface_fade_out()`, or `display_screen()`
- Calls: `get_screen_data()`, `interface_fade_out()`, `display_screen()`, `images_picture_exists()`
- Notes: If screen resource doesn't exist, sets phase to 0 (skip it).

### display_screen
- Signature: `void display_screen(short base_pict_id)`
- Purpose: Render a picture resource to screen with fade-in; set up color table and prepare for next-screen timeout.
- Inputs: `base_pict_id` (resource ID base; current screen is offset by `game_state.current_screen`)
- Outputs/Return: None
- Side effects: Allocates/deletes `current_picture_clut`, calls `full_fade()` and `start_interface_fade()`, draws resource
- Calls: `calculate_picture_clut()`, `assert_world_color_table()`, `full_fade()`, `draw_full_screen_pict_resource_from_images()`, `start_interface_fade()`

### begin_game
- Signature: `bool begin_game(short user, bool cheat)`
- Purpose: Initialize a new game for single-player, network, or demo play; set up world, players, and recording.
- Inputs: `user` (player type: single, network, demo), `cheat` (whether cheats enabled)
- Outputs/Return: `true` if successful
- Side effects: Calls `allocate_game_memory()`, `generate_map()`, `initialize_players()`, `start_recording()` if needed
- Calls: Many game-world functions (map gen, player init, physics, recording, network setup)
- Notes: Enforces standard behavior for network and film modes.

### finish_game
- Signature: `void finish_game(bool return_to_main_menu)`
- Purpose: Clean up after game endΓÇöstop recording, close resources, display stats if networked.
- Inputs: `return_to_main_menu` (true ΓåÆ show menu; false ΓåÆ show quit screens)
- Outputs/Return: None
- Side effects: Stops recording, exits networking, calls `display_net_game_stats()` if networked
- Calls: `stop_recording()`, `display_net_game_stats()`, `exit_networking()`, `display_main_menu()`, `display_quit_screens()`

### handle_interface_menu_screen_click
- Signature: `void handle_interface_menu_screen_click(short x, short y, bool cheatkeys_down)`
- Purpose: Process mouse clicks on main menu buttons; show press feedback and dispatch menu commands.
- Inputs: `x`, `y` (click coordinates), `cheatkeys_down` (whether cheat modifiers held)
- Outputs/Return: None
- Side effects: Calls `draw_button()`, `do_menu_item_command()` when button released over a valid rectangle
- Calls: `point_in_rectangle()`, `get_interface_rectangle()`, `draw_button()`, `do_menu_item_command()`
- Notes: Handles click-through during movie playback by early-returning; clips coordinates by SDL video surface offsets.

### interface_fade_out
- Signature: `void interface_fade_out(short pict_resource_number, bool fade_music)`
- Purpose: Fade out screen to black before major transition (e.g., entering gameplay); optionally fade music.
- Inputs: `pict_resource_number` (resource for color table), `fade_music` (fade music volume)
- Outputs/Return: None
- Side effects: Allocates animated color table, calls `explicit_start_fade()`, optionally calls `Music::FadeOut()`, deletes color table when done
- Calls: `calculate_picture_clut()`, `explicit_start_fade()`, `update_fades()`, `Music::instance()->FadeOut()`, `paint_window_black()`
- Notes: Loops until fade completes; recalculates color table if bit depth changed mid-fade.

### construct_single_player_start
- Signature: `void construct_single_player_start(player_start_data* outStartArray, short* outStartCount)`
- Purpose: Build a single-player game start from player preferences.
- Inputs: `outStartArray` (destination), `outStartCount` (pointer to count)
- Outputs/Return: None; fills array and sets count
- Side effects: Populates start with player color, name, and behavior flags
- Calls: None (reads from `player_preferences`)

### synchronize_players_with_starts
- Signature: `static void synchronize_players_with_starts(const player_start_data* inStartArray, short inStartCount)`
- Purpose: Update existing players' appearance/state or create new players to match a start array (used when resuming or joining).
- Inputs: `inStartArray` (player starts), `inStartCount` (count)
- Outputs/Return: None
- Side effects: Modifies player data (team, color, name, flags); marks unmatched players as zombies; calls `new_player()` for extras
- Calls: `get_player_data()`, `SET_PLAYER_ZOMBIE_STATUS()`, `new_player()`

### match_starts_with_existing_players
- Signature: `void match_starts_with_existing_players(player_start_data* ioStartArray, short* ioStartCount)`
- Purpose: Reorder and extend start array to align with existing player roster (by name, then arbitrarily).
- Inputs: `ioStartArray` (in/out), `ioStartCount` (in/out)
- Outputs/Return: None; modifies array in place
- Side effects: Reorders starts, creates new starts for unmatched players, sets zombies for leftover players
- Calls: `strcmp()`, `strcpy()`, `get_player_data()`

### make_restored_game_relevant
- Signature: `static bool make_restored_game_relevant(bool inNetgame, const player_start_data* inStartArray, short inStartCount)`
- Purpose: Restore a saved game to play stateΓÇöset random seed, sync players, configure world, enter map.
- Inputs: `inNetgame` (is this a network game), `inStartArray`, `inStartCount`
- Outputs/Return: `true` if successful, `false` if map load failed
- Side effects: Calls `set_random_seed()`, `synchronize_players_with_starts()`, `entering_map()`, `reset_motion_sensor()`
- Calls: `set_random_seed()`, `synchronize_players_with_starts()`, `entering_map()`, `reset_motion_sensor()`, `clean_up_after_failed_game()`

### should_restore_game_networked
- Signature: `size_t should_restore_game_networked()`
- Purpose: Show dialog asking whether to resume single or networked; used after loading save file.
- Inputs: None
- Outputs/Return: `UNONE` (cancel), `0` (single-player), `1` (networked)
- Side effects: Displays modal dialog with toggle widget
- Calls: `dialog` class and widget construction

### show_movie
- Signature: `void show_movie(short index)`
- Purpose: Play a movie file (intro or level-specific) using SMPEG; respond to user input to skip.
- Inputs: `index` (movie index, 0 = intro)
- Outputs/Return: None
- Side effects: Pauses sound, creates SMPEG instance, renders to SDL surface, deletes movie on completion
- Calls: `GetLevelMovie()`, `SMPEG_new()`, `SMPEG_setdisplay()`, `SMPEG_play()`, `SDL_PollEvent()`
- Notes: Only compiled if `HAVE_SMPEG` defined; scales video and centers on screen.

### handle_network_game
- Signature: `static void handle_network_game(bool gatherer)`
- Purpose: Orchestrate network game gathering (host) or joining; transition to gameplay or error recovery.
- Inputs: `gatherer` (true = host, false = joiner)
- Outputs/Return: None
- Side effects: Sets state to `_displaying_network_game_dialogs`, calls `network_gather()`, `NetStart()`, or `network_join()`
- Calls: `force_system_colors()`, `network_gather()`, `NetStart()`, `network_join()`, `join_networked_resume_game()`, `begin_game()`, `display_main_menu()`

### join_networked_resume_game
- Signature: `bool join_networked_resume_game()`
- Purpose: Receive and restore a saved game that the gatherer is sharing for network play.
- Inputs: None
- Outputs/Return: `true` if successful
- Side effects: Receives game data, deserializes, calls `make_restored_game_relevant()`
- Calls: `NetReceiveGameData()`, `NetGetGameData()`, `unpack_dynamic_data()`, `make_restored_game_relevant()`, `clean_up_after_failed_game()`

## Control Flow Notes

**Initialization ΓåÆ Menu Loop ΓåÆ Game ΓåÆ Cleanup:**

1. **Boot**: `initialize_game_state()` ΓåÆ state = `_display_intro_screens`, calls `display_introduction()`
2. **Main idle loop** (`idle_game_state()`):
   - Dispatches on `game_state.state`
   - For intro/screen sequences: increments phase timer, calls `next_game_screen()` when duration elapsed
   - For `_game_in_progress`: calls `update_world()`, `update_players()`, `NetSync()`
   - Updates active fades via `update_interface_fades()`
3. **State transitions** via `set_game_state()`: validates and defers network-sensitive ops
4. **Game end**: `finish_game()` ΓåÆ cleanup, show stats, return to menu
5. **Fades** orchestrate color table transitions around major state changes

**Network flow:**
- `handle_network_game(true)` ΓåÆ `network_gather()` ΓåÆ `NetStart()` ΓåÆ `begin_game()`
- `handle_network_game(false)` ΓåÆ `network_join()` ΓåÆ `join_networked_resume_game()` or `begin_game()`

**Resume flow:**
- Load saved game ΓåÆ `should_restore_game_networked()` dialog ΓåÆ `make_restored_game_relevant()` ΓåÆ `entering_map(true)`

## External Dependencies

- **World/Game**: `map.h`, `player.h` (player data/init), `dynamic_world`, `entering_map()`, `update_world()`, `update_players()`
- **Rendering**: `screen_drawing.h`, `render.h`, `OGL_Render.h`, `images.h` (picture resources)
- **Sound/Music**: `SoundManager.h`, `Music.h` (music fading), `network_sound.h` (netmic)
- **Networking**: `network.h`, `network_dialog_widgets_sdl.h` (UI), `NetGetNumberOfPlayers()`, `NetSync()`, `NetStart()`, etc.
- **UI/Dialogs**: `interface_menus.h`, `sdl_dialogs.h`, `sdl_widgets.h`
- **Scripting**: `lua_script.h` (PostIdle), `XML_LevelScript.h` (end-game script)
- **File I/O**: `FileHandler.h` (`FileSpecifier`)
- **Video**: `smpeg/smpeg.h` (movies, conditional)
