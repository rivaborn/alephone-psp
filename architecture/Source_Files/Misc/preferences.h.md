# Source_Files/Misc/preferences.h

## File Purpose
Declares data structures and functions for managing persistent game preferences. Covers graphics configuration, network settings, player data, input controls, environment file paths, and serial number validation. Serves as the central definition for all preference-related state in the Aleph One engine.

## Core Responsibilities
- Define preference data structures for six major subsystems (graphics, network, player, input, environment, serial)
- Declare extern pointers to global preference instances
- Declare preference lifecycle functions (initialize, read, write, handle)
- Define enums and constants for preference options (blending modes, network protocols, input modifiers, shell keys)
- Aggregate and expose OpenGL, chase-cam, crosshair, and sound configuration via nested structures

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `graphics_preferences_data` | struct | Screen mode, OpenGL configuration, CPU affinity, corpse limits, alpha blending settings |
| `network_preferences_data` | struct | Game type, difficulty, time/kill limits, microphone, metaserver login, netscript, UPnP, cheat flags |
| `player_preferences_data` | struct | Player name, color, team, last play time, difficulty, music toggle, chase-cam and crosshair data |
| `input_preferences_data` | struct | Input device, key bindings (gameplay and shell), mouse sensitivity, button action mapping, modifier flags |
| `environment_preferences_data` | struct | File paths to map/physics/shapes/sounds, checksums, patch list, Lua script settings, theme/display options |
| `serial_number_data` | struct | Network-only flag, serial number bytes, user name, tokenized serial |
| `screen_mode_data` | struct | Resolution, acceleration, fullscreen, bit depth, gamma (from shell.h) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `graphics_preferences` | `graphics_preferences_data*` | extern global | Active graphics configuration |
| `serial_preferences` | `serial_number_data*` | extern global | Active serial number and registration data |
| `network_preferences` | `network_preferences_data*` | extern global | Active network game settings |
| `player_preferences` | `player_preferences_data*` | extern global | Active player profile (name, color, team, etc.) |
| `input_preferences` | `input_preferences_data*` | extern global | Active input device and key bindings |
| `sound_preferences` | `SoundManager::Parameters*` | extern global | Active sound configuration (volume, channels, rate) |
| `environment_preferences` | `environment_preferences_data*` | extern global | Active environment file paths and Lua settings |

## Key Functions / Methods

### initialize_preferences
- Signature: `void initialize_preferences(void)`
- Purpose: Initialize preference system at engine startup
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/initializes global preference pointers; loads defaults
- Calls: (implementation in preferences.cpp, not visible here)
- Notes: Must be called before any preference access

### read_preferences
- Signature: `void read_preferences()`
- Purpose: Load preferences from persistent storage (file/registry)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates all extern preference pointers with loaded values
- Calls: (implementation in preferences.cpp)
- Notes: Typically called during game startup after initialization

### write_preferences
- Signature: `void write_preferences()`
- Purpose: Save current preference state to persistent storage
- Inputs: None
- Outputs/Return: None
- Side effects: Writes all global preference structures to disk/registry
- Calls: (implementation in preferences.cpp)
- Notes: Called on config changes or game shutdown

### handle_preferences
- Signature: `void handle_preferences(void)`
- Purpose: Trigger preference dialog UI and apply user changes
- Inputs: None
- Outputs/Return: None
- Side effects: May update any/all preference structures if user confirms changes
- Calls: (implementation in preferences.cpp, likely calls graphics/network/input dialog handlers)
- Notes: Entry point for user-driven preference modification

## Control Flow Notes
This header is part of the **initialization and persistent state** subsystem. The preference lifecycle is:
1. `initialize_preferences()` ΓÇö engine startup
2. `read_preferences()` ΓÇö load saved settings
3. During gameplay: `handle_preferences()` on user request (menu interaction)
4. On shutdown or config change: `write_preferences()` to persist

Preferences are queried throughout the engine via the extern pointers (e.g., `graphics_preferences->OGL_Configure` in rendering code, `input_preferences->keycodes` in input handling). The architecture uses pointer indirection to allow runtime preference updates without recompilation.

## External Dependencies
- **interface.h** ΓÇö game state and UI constants (`NUMBER_OF_KEY_SETUPS`, etc.); dialog definitions
- **ChaseCam.h** ΓÇö `ChaseCamData` struct (nested in `player_preferences_data`)
- **Crosshairs.h** ΓÇö `CrosshairData` struct (nested in `player_preferences_data`)
- **OGL_Setup.h** ΓÇö `OGL_ConfigureData` struct (nested in `graphics_preferences_data`)
- **shell.h** ΓÇö `screen_mode_data` struct (nested in `graphics_preferences_data`); `NUMBER_OF_KEYS`, `PREFERENCES_VERSION`, etc.
- **SoundManager.h** ΓÇö `SoundManager::Parameters` struct (extern as `sound_preferences`)
- Standard C headers (implicit via includes)
