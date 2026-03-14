# Source_Files/shell.h

## File Purpose

Core header for the Aleph One game engine's cross-platform shell layer. Defines constants, structures, and function prototypes for UI management, input configuration, display modes, preferences, resource loading (shapes/MML), and platform-specific functionality (Mac/SDL/PSP support).

## Core Responsibilities

- **Window and dialog reference management** ΓÇô Enums and constants for screen windows and preference/configuration dialogs
- **Display mode configuration** ΓÇô Structures defining resolution, bit depth, fullscreen, acceleration, gamma, and frame-skip settings
- **Input device enumeration** ΓÇô Support for keyboard, mouse (yaw/pitch/velocity), game pads, CyberMaxx, and Input Sprocket
- **System capability detection** ΓÇô Flags for OS version, processor type, network availability, QuickTime, etc.
- **MML/XML resource loading** ΓÇô Integration with Loren Petrich's XML parser for cheat codes and configuration
- **Shape/texture access** ΓÇô Platform-specific functions for loading and retrieving sprite/bitmap data (Mac/SDL)
- **Preferences persistence** ΓÇô Constants and functions for saving/loading user settings
- **Event handling** ΓÇô Platform-specific key input and window update handlers
- **Screen text output** ΓÇô Printf-style text rendering to the game display

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `screen_mode_data` | struct | Display configuration: resolution, bit depth, fullscreen, gamma, acceleration, frame-skip flags |
| `system_information_data` | struct | Capability flags: OS version (7/10), processors (68k/PPC), network/quicktime/appleevents availability |
| Window/Dialog ref enums | enum | Constants for referencing screen windows (1000) and dialogs (8000ΓÇô8005, preferences/network/keyboard config) |
| Input device enum | enum | Device selection: keyboard, mouse yaw/pitch, mouse velocity, CyberMaxx, Input Sprocket |
| Sound ref constant | #define | `sndCHANGED_VOLUME_SOUND` (2000) |
| Prompt string resources | enum | String resource indices for save/load/replay prompts |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `system_information` | `struct system_information_data*` | global | Platform capabilities detected at startup |
| `MainBundle` | `CFBundleRef` | global | Mac/Carbon: Main application bundle reference |
| `GUI_Nib` | `IBNibRef` | global | Mac/Carbon: GUI resource bundle (NIB) |
| `psp_mouse` | `PSPSDLMouse*` | global | PSP: Simulated mouse instance for analog input |

## Key Functions / Methods

### global_idle_proc
- Signature: `void global_idle_proc(void)`
- Purpose: Main idle/frame-update loop called continuously during gameplay
- Inputs: None
- Outputs/Return: None
- Side effects: Updates game state, input, rendering; drives the main game loop
- Calls: (Implementation in shell_misc.cpp, shell_macintosh.cpp, or shell_sdl.cpp)
- Notes: Core event loop; defined elsewhere based on platform

### handle_game_key (Mac only)
- Signature: `void handle_game_key(EventRecord *event, short key)`
- Purpose: Dispatch game-relevant key events from the OS event queue
- Inputs: Mac EventRecord, key code
- Outputs/Return: None
- Side effects: Updates game input state or triggers actions
- Calls: (Implementation in shell_macintosh.cpp)

### LoadBaseMMLScripts
- Signature: `void LoadBaseMMLScripts(void)`
- Purpose: Load default Marathon Markup Language configuration scripts at startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes cheat codes, weapons, entities from MML resource files

### initialize_shape_handler
- Signature: `void initialize_shape_handler(void)`
- Purpose: Initialize the shape/sprite caching and rendering system
- Inputs: None
- Outputs/Return: None
- Side effects: Sets up texture/bitmap management

### get_shape_surface (SDL only)
- Signature: `SDL_Surface *get_shape_surface(int shape, int collection = NONE, byte** outPointerToPixelData = NULL, float inIllumination = -1.0f, bool inShrinkImage = false)`
- Purpose: Retrieve an SDL surface for a sprite shape, optionally with shading/illumination or shrinking
- Inputs: Shape ID, collection ID, illumination level, shrink flag
- Outputs/Return: Allocated SDL_Surface; optional pixel data pointer for RLE shapes
- Side effects: Allocates memory; caller must free SDL_Surface
- Notes: Handles RLE-encoded shapes and team/player colorization; caller must free outPointerToPixelData if using RLE

### open_shapes_file
- Signature: `void open_shapes_file(FileSpecifier& File)`
- Purpose: Load a shapes/sprite resource file
- Inputs: File path
- Outputs/Return: None
- Side effects: Loads shape data into engine

### load_environment_from_preferences
- Signature: `void load_environment_from_preferences(void)`
- Purpose: Load saved preferences (display mode, input config, etc.) at startup
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes display/input from stored settings

### screen_printf
- Signature: `void screen_printf(const char *format, ...)`
- Purpose: Printf-style text output to the game screen
- Inputs: Format string and arguments
- Outputs/Return: None
- Side effects: Renders text overlay on display
- Notes: Implemented in screen routines

**Helper/Trivial Functions:**
- `machine_has_quicktime()`, `machine_has_nav_services()` ΓÇô macOS capability checks
- `GetSystemFunctionPointer()` ΓÇô macOS/Carbon dynamic function resolution
- `XML_LoadFromResourceFork()` ΓÇô Load MML from Mac resource fork
- `Cheats_GetParser()` ΓÇô Retrieve XML parser for cheat definitions
- Color getters: `_get_player_color()`, `_get_interface_color()` ΓÇô Retrieve RGB colors for UI/players
- Event/window handlers: `process_game_key()`, `update_game_window()`, `has_cheat_modifiers()` ΓÇô Input/display management

## Control Flow Notes

This header orchestrates **initialization ΓåÆ main loop ΓåÆ shutdown**:

1. **Init phase:** `initialize_shape_handler()` ΓåÆ `load_environment_from_preferences()` ΓåÆ `LoadBaseMMLScripts()` ΓåÆ `XML_LoadFromResourceFork()` (if available)
2. **Frame loop:** `global_idle_proc()` called repeatedly; internally handles input via `handle_game_key()`, updates display via `update_game_window()`
3. **Resource access:** Game accesses sprites via `get_shape_surface()`, colors via `_get_player_color()` / `_get_interface_color()`
4. **Debug/UI:** `screen_printf()` for on-screen text

Strongly platform-dependent (Mac/SDL/PSP conditional compilation) but maintains consistent interface across targets.

## External Dependencies

- **XML_ElementParser.h** ΓÇô Custom XML/MML parsing infrastructure (Loren Petrich)
- **Input/psp_mouse_sdl.h** ΓÇô PSP-specific mouse simulation (conditional on `#ifdef PSP`)
- **FileSpecifier** ΓÇô Forward-declared; file path abstraction (defined elsewhere)
- **Mac/Carbon frameworks** ΓÇô CFBundle, IBNib (conditional on `#ifdef TARGET_API_MAC_CARBON`)
- **SDL** ΓÇô Cross-platform graphics/input (conditional on `#ifdef SDL`)
- **EventRecord** ΓÇô Mac OS event structure (conditional on `#ifdef mac`)
- **Color types:** `RGBColor` (Mac), `SDL_Color` (SDL) ΓÇô platform-specific color representations
