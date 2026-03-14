# Source_Files/Files/wad_prefs.h

## File Purpose
Provides interfaces for managing WAD (game data) preference files, including opening, reading, validating, and writing preferences to persistent storage. Supports both programmatic access and macOS-specific dialog-based preference UI configuration.

## Core Responsibilities
- Define function signatures for preference file I/O (open, read, write)
- Declare callback function pointer types for preference initialization and validation
- Manage internal preference data storage and file references
- Provide macOS-specific dialog UI structure for interactive preference configuration

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `prefs_initializer` | typedef | Function pointer: `void(*)(void *prefs)` ΓÇô called to allocate/initialize preference data |
| `prefs_validater` | typedef | Function pointer: `bool(*)(void *prefs)` ΓÇô called to validate and repair preference data |
| `preferences_info` | struct | Holds FileSpecifier for preferences file and pointer to wad_data blob |
| `preferences_dialog_data` | struct | MacOS-only; configures preference dialog UI (resource IDs, callbacks for setup/teardown/item events) |

## Global / File-Static State
None.

## Key Functions / Methods

### w_open_preferences_file
- Signature: `bool w_open_preferences_file(char *PrefName, Typecode Type)`
- Purpose: Opens a preferences file and allocates internal structures
- Inputs: Preference filename, file type code
- Outputs/Return: True if successful
- Side effects: Allocates and initializes internal preference data structures
- Calls: (Not inferable from this file)
- Notes: Called during initialization; filename and type specify which preferences to load

### w_get_data_from_preferences
- Signature: `void *w_get_data_from_preferences(WadDataType tag, size_t expected_size, prefs_initializer initialize, prefs_validater validate)`
- Purpose: Retrieves typed preference data, with automatic initialization and validation
- Inputs: Tag (identifier), expected data size, initialization callback, validation callback
- Outputs/Return: Pointer to preference data (caller must not free)
- Side effects: Allocates preference data if missing; calls initialize/validate callbacks
- Calls: User-provided callbacks (initialize, validate)
- Notes: Returns existing data if present; calls initialize() only if allocation needed; calls validate() to verify/repair data

### w_write_preferences_file
- Signature: `void w_write_preferences_file(void)`
- Purpose: Persists all loaded preferences to disk
- Inputs: None
- Outputs/Return: None
- Side effects: Writes preference data to file
- Calls: (Not inferable from this file)
- Notes: No error reporting; assumes file operations succeed

### set_preferences (macOS only)
- Signature: `bool set_preferences(struct preferences_dialog_data *funcs, short count, void (*reload_function)(void))`
- Purpose: Runs a macOS preference dialog with custom UI layout and callbacks
- Inputs: Array of dialog configurations, count, reload callback
- Outputs/Return: True if user confirmed dialog
- Side effects: Shows modal dialog; calls setup/teardown/item_hit callbacks; calls reload_function on completion
- Calls: User-provided callbacks (setup_dialog_func, item_hit_func, teardown_dialog_func, reload_function)
- Notes: Macintosh-specific; encapsulates classic Mac dialog item management

## Control Flow Notes
1. **Initialization**: `w_open_preferences_file()` opens and initializes the preference file
2. **Data access**: `w_get_data_from_preferences()` retrieves typed data blocks, validating/initializing as needed
3. **UI (macOS)**: `set_preferences()` runs an interactive dialog to edit preferences
4. **Persistence**: `w_write_preferences_file()` writes modified data back to disk

Not inferable: where this fits into engine startup/shutdown or frame updates.

## External Dependencies
- **FileHandler.h**: `FileSpecifier` (file path abstraction), `OpenedResourceFile` (resource I/O)
- **Tags.h** (via FileHandler.h): `Typecode` enum for file types
- **Undefined in this file**: `WadDataType`, `struct wad_data`, `w_get_data_from_preferences()` implementation
