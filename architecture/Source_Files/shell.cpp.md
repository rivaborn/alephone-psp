# Source_Files/shell.cpp

## File Purpose
Main game shell, serving as the entry point and central event loop for the Aleph One game engine. Handles SDL initialization, command-line parsing, input event dispatching, game state transitions, and graceful shutdown across multiple platforms (Windows, macOS, Linux, PSP).

## Core Responsibilities
- **Application lifecycle**: `main()`, `initialize_application()`, `shutdown_application()`
- **Main event loop**: Poll SDL events, dispatch input, update game state, maintain frame timing
- **Input routing**: Keyboard (game keys, menu keys, function keys), mouse clicks, system events
- **Platform setup**: Platform-specific initialization (PSP callbacks, SDL configuration, directories)
- **Data directories**: Locate and construct search paths for map, sound, shape, preference files
- **Game state machine**: Transition between intro screens, menus, active gameplay, demos
- **Configuration**: Parse command-line arguments (fullscreen, sound, OpenGL, debug mode)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| screen_mode_data | struct | Graphics settings (resolution, acceleration, gamma) |
| view_data | struct | Rendering parameters (FOV, screen dimensions, viewport) |
| game_data | struct | Game type, difficulty, time limit, parameters |
| player_data | struct | Player state (location, energy, weapons, inventory) |
| player_settings_definition | struct | Configurable player mechanics (energy, oxygen, vision) |
| SoundManager::Parameters | struct | Audio configuration (volume, sample rate, channels) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| data_search_path | vector<DirectorySpecifier> | global | Ordered list of directories to search for game data |
| local_data_dir | DirectorySpecifier | global | Per-user local data directory (prefs, saves, recordings) |
| preferences_dir | DirectorySpecifier | global | User preferences directory |
| saved_games_dir | DirectorySpecifier | global | Saved game files directory |
| recordings_dir | DirectorySpecifier | global | Film/replay recordings directory |
| vidmasterStringSetID | short | file-static | Custom Vidmaster oath string set ID (from MML) |
| option_nogl | bool | file-static | Disable OpenGL flag |
| option_nosound | bool | file-static | Disable sound output flag |
| option_nogamma | bool | file-static | Disable gamma/menu fades flag |
| option_debug | bool | file-static | Debug/core-dump mode flag |
| force_fullscreen | bool | file-static | Force fullscreen mode override |
| force_windowed | bool | file-static | Force windowed mode override |
| psp_mouse | PSPSDLMouse* | file-static | PSP platform mouse simulator |
| arg_directory | string | global | Command-line specified data directory |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char **argv)`
- **Purpose:** Entry point; parses arguments, initializes engine, runs main loop, handles exceptions
- **Inputs:** Command-line arguments (`argc`, `argv`)
- **Outputs/Return:** Exit code (0 on success, 1 on exception)
- **Side effects:** Initializes all engine systems; registers `atexit(shutdown_application)`; calls `SDL_Init()`
- **Calls:** `initialize_application()`, `main_event_loop()`, exception handlers
- **Notes:** Exits immediately on unrecognized arguments or initialization failure; PSP-specific callback setup

### initialize_application
- **Signature:** `static void initialize_application(void)`
- **Purpose:** One-time engine initialization: locate data files, initialize SDL/audio/video subsystems, load MML scripts, initialize all game systems
- **Inputs:** None (uses global state and preferences files)
- **Outputs/Return:** None (modifies global state)
- **Side effects:** Creates directories; initializes SDL, SoundManager, rendering, fonts, MML parser, physics, screen drawing, shape handler, fades, images, game state
- **Calls:** `SDL_Init()`, `initialize_resources()`, `initialize_fonts()`, `SetupParseTree()`, `LoadBaseMMLScripts()`, `initialize_preferences()`, `initialize_keyboard_controller()`, `initialize_screen()`, `initialize_marathon()`, numerous subsystem initializers
- **Notes:** Platform-specific directory setup (Unix, macOS, BeOS, Windows, PSP); aborts if required text strings missing; applies force_fullscreen/force_windowed overrides

### shutdown_application
- **Signature:** `static void shutdown_application(void)`
- **Purpose:** Graceful cleanup; tears down SDL, PSP, and platform-specific resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Quits SDL_sound, SDL_net, SDL_ttf, SDL; deletes psp_mouse; calls PSP kernel shutdown
- **Calls:** `SDLNet_Quit()`, `Sound_Quit()`, `TTF_Quit()`, `SDL_Quit()`, `sceKernelSleepThread()`, `sceKernelExitGame()`
- **Notes:** Protects against recursive shutdown with `already_shutting_down` flag; logs shutdown stages on PSP

### main_event_loop
- **Signature:** `static void main_event_loop(void)`
- **Purpose:** Central game loop; polls events, updates game state, manages frame timing
- **Inputs:** None (reads global game state)
- **Outputs/Return:** None (loop exits when `get_game_state() == _quit_game`)
- **Side effects:** Calls `process_event()`, `execute_timer_tasks()`, `idle_game_state()` each frame; may call `SDL_Delay()` for CPU throttling
- **Calls:** `get_game_state()`, `SDL_GetTicks()`, `SDL_PumpEvents()`, `SDL_PollEvent()`, `SDL_Delay()`, `process_event()`, `execute_timer_tasks()`, `idle_game_state()`, `global_idle_proc()`
- **Notes:** Switches polling frequency based on game state (6 Hz baseline for most UI, responsive keyboard polling in gameplay); PSP mouse update occurs before event dispatch; CPU throttling applied in non-hot states

### process_event
- **Signature:** `static void process_event(const SDL_Event &event)`
- **Purpose:** Main event dispatcher; routes SDL events to appropriate handlers based on type
- **Inputs:** SDL_Event reference
- **Outputs/Return:** None
- **Side effects:** Updates keyboard controller status, game state, console input; may darken/redraw screen
- **Calls:** `process_screen_click()`, `process_game_key()`, `process_system_event()`, `set_game_state()`, `darken_world_window()`, `show_cursor()`, `hide_cursor()`, `set_keyboard_controller_status()`, `update_game_window()`, `SDL_GL_SwapBuffers()`
- **Notes:** Handles SDL_MOUSEBUTTONDOWN (scroll detection, cursor hide), SDL_KEYDOWN, SDL_SYSWMEVENT, SDL_QUIT, SDL_ACTIVEEVENT (pause on focus loss), SDL_VIDEOEXPOSE

### process_game_key
- **Signature:** `static void process_game_key(const SDL_Event &event)`
- **Purpose:** Game state-specific keyboard handler; dispatches to menu handlers or gameplay key handler based on current state
- **Inputs:** SDL_Event (key press)
- **Outputs/Return:** None
- **Side effects:** May change game state, trigger menu commands, modify graphics/interface settings
- **Calls:** `handle_game_key()`, `do_menu_item_command()`, `force_game_state_change()`, `stop_interface_fade()`, `display_main_menu()`, `draw_menu_button_for_command()`
- **Notes:** Distinguishes macOS (Command key) from Windows/Linux (Alt key) for menu shortcuts; handles Alt+F4 in Windows; supports Escape to force state changes during intro screens

### handle_game_key
- **Signature:** `static void handle_game_key(const SDL_Event &event)`
- **Purpose:** Gameplay key handling; processes volume, zoom, console, visual modes (tunnel vision, crosshairs, chase cam), gamma, screen size, and special keys
- **Inputs:** SDL_Event (key press); reads modifiers and current console/game state
- **Outputs/Return:** None
- **Side effects:** Updates graphics_preferences, SoundManager volume, Console input, Crosshairs_IsActive, ChaseCam settings; may change screen mode and write preferences
- **Calls:** `SoundManager::instance()->AdjustVolumeUp/Down()`, `Console::instance()` methods, `PlayInterfaceButtonSound()`, `render_screen()`, `zoom_overhead_map_in/out()`, `scroll_inventory()`, `change_gamma_level()`, `change_screen_mode()`, `OGL_ResetTextures()`, `dump_screen()`, `toggle_fullscreen()`, `do_menu_item_command()`
- **Notes:** F1-F12 reserved for engine features (screen size, gamma, OpenGL reset, chase cam, tunnel vision, crosshairs, screenshots); PSP-specific handling of PSP_CTRL_START button; cheat activation restricted to single-player, Ctrl held, and CheatsActive flag

### process_screen_click
- **Signature:** `static void process_screen_click(const SDL_Event &event)`
- **Purpose:** Route mouse clicks to portable screen click handler with cheat modifier detection
- **Inputs:** SDL_Event (mouse button down)
- **Outputs/Return:** None
- **Side effects:** May interact with interface/menu items
- **Calls:** `portable_process_screen_click()`, `has_cheat_modifiers()`

### dump_screen
- **Signature:** `void dump_screen(void)`
- **Purpose:** Capture current screen to BMP file; incremental numbering to avoid overwrites
- **Inputs:** None (reads video surface and OpenGL frame buffer)
- **Outputs/Return:** None (writes .bmp file to local_data_dir)
- **Side effects:** Creates `Screenshot_XXXX.bmp` file; allocates temporary SDL surface and pixel buffer for OpenGL
- **Calls:** `SDL_GetVideoSurface()`, `SDL_SaveBMP()`, `glReadPixels()`, `SDL_CreateRGBSurface()`, `SDL_FreeSurface()`, `malloc()`, `free()`
- **Notes:** OpenGL screenshots are Y-flipped during pixel copy; endianness-aware RGB format selection

### LoadBaseMMLScripts
- **Signature:** `void LoadBaseMMLScripts()`
- **Purpose:** Recursively load and parse MML (Marathon Markup Language) scripts from data search path
- **Inputs:** None (uses global data_search_path)
- **Outputs/Return:** None
- **Side effects:** Populates XML parse tree with MML definitions from "MML" and "Scripts" subdirectories
- **Calls:** `XML_Loader_SDL::ParseDirectory()` for each data search directory + "MML" and "Scripts" subdirectory
- **Notes:** Executes for each directory in data_search_path, allowing modular script organization

### PlayInterfaceButtonSound
- **Signature:** `void PlayInterfaceButtonSound(short SoundID)`
- **Purpose:** Utility to play interface button feedback sound if enabled in input preferences
- **Inputs:** SoundID index
- **Outputs/Return:** None
- **Side effects:** Plays sound via SoundManager (or suppressed if muted in preferences)
- **Calls:** `TEST_FLAG()`, `SoundManager::instance()->PlaySound()`
- **Notes:** Checks input_preferences->modifiers._inputmod_use_button_sounds; passes NULL location for non-spatial sound

## Control Flow Notes
**Initialization ΓåÆ Game Loop ΓåÆ Shutdown**
1. `main()` parses arguments and calls `initialize_application()` (one-time setup)
2. `initialize_application()` initializes all subsystems: SDL, audio, rendering, MML parsing, game state
3. `main_event_loop()` runs continuously, polling events and updating game state until `_quit_game` is set
4. Event loop dispatches SDL events: keyboard ΓåÆ `process_game_key()` ΓåÆ `handle_game_key()` or menu handlers; mouse ΓåÆ `process_screen_click()`; system ΓåÆ `process_system_event()` (Windows DirectShow)
5. Each frame: `execute_timer_tasks()` updates time-driven logic, `idle_game_state()` processes gameplay
6. On exit or exception, `atexit()` triggers `shutdown_application()` for cleanup

**Game State Transitions:**
- Menus: `_display_main_menu`, `_display_intro_screens`, etc. ΓåÆ fade timing ΓåÆ key/click input forces transition
- Gameplay: `_game_in_progress` ΓåÆ `process_game_key()` ΓåÆ ESC or Alt+Q quits; F-keys adjust view/audio
- Level change: `_change_level` ΓåÆ `idle_game_state()` handles transition
- Exit: ESC or quit dialog sets `_quit_game` ΓåÆ loop exits

## External Dependencies
- **Notable includes**: cseries.h (base types, macros), map.h (world data), monsters.h, player.h, render.h, interface.h, SoundManager.h, Crosshairs.h, OGL_Render.h, XML_ParseTreeRoot.h, FileHandler.h
- **SDL**: SDL, SDL_net, SDL_sound, SDL_ttf, SDL_syswm
- **Platform-specific**: windows.h (Win32), pspkernel.h/pspdebug.h/pspctrl.h (PSP), CoreFoundation/OpenGL (macOS)
- **External symbols defined elsewhere**: 
  - `initialize_marathon()`, `initialize_screen()`, `initialize_keyboard_controller()`, `initialize_screen_drawing()`, `initialize_game_state()` (lifecycle)
  - `idle_game_state()`, `render_screen()`, `update_game_window()` (rendering)
  - `SoundManager::instance()`, `Music::instance()`, `Console::instance()` (audio/console)
  - `Crosshairs_IsActive()`, `ChaseCam_IsActive()` (visual modes)
  - `process_keyword_key()`, `handle_keyword()` (cheat system)
  - `do_menu_item_command()`, `force_game_state_change()` (menu/state)
  - `portable_process_screen_click()` (input)
