# Source_Files/Files/import_definitions.cpp

## File Purpose

Manages loading and importing of physics model definitions (monsters, effects, projectiles, weapons, physics constants) for the game engine. Handles both loading from local WAD files and processing physics data transmitted over the network for netgame synchronization.

## Core Responsibilities

- Maintain the current physics file specification and provide accessors for setting/resetting it
- Initialize all physics model data structures during game startup
- Load physics definitions from disk-based WAD physics files
- Extract and unpack individual physics definition types from WAD data
- Serialize physics data for network transmission to clients
- Deserialize and apply physics data received from network

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | class (defined elsewhere) | Represents a physics file path/location |
| `OpenedFile` | class (defined elsewhere) | Handle to an opened WAD file |
| `wad_data` | struct | Container for unpacked WAD data with tag entries |
| `wad_header` | struct | WAD file header with metadata and checksums |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PhysicsFileSpec` | `FileSpecifier` | static | Stores the current physics file path; updated by `set_physics_file()` |

## Key Functions / Methods

### set_physics_file
- Signature: `void set_physics_file(FileSpecifier& File)`
- Purpose: Set the physics file to load from
- Inputs: `File` ΓÇô a FileSpecifier pointing to the desired physics file
- Outputs/Return: None
- Side effects: Updates `PhysicsFileSpec` (global state)
- Notes: Does not validate or open the file; just stores the path for later use

### set_to_default_physics_file
- Signature: `void set_to_default_physics_file(void)`
- Purpose: Reset physics file to the engine's default location
- Inputs: None
- Outputs/Return: None
- Side effects: Updates `PhysicsFileSpec` by calling `get_default_physics_spec()`
- Calls: `get_default_physics_spec()` (defined elsewhere)

### init_physics_wad_data
- Signature: `void init_physics_wad_data()`
- Purpose: Initialize all physics model definitions to default/empty state
- Inputs: None
- Outputs/Return: None
- Side effects: Calls initialization functions for all physics subsystems
- Calls: `init_monster_definitions()`, `init_effect_definitions()`, `init_projectile_definitions()`, `init_physics_constants()`, `init_weapon_definitions()`
- Notes: Called at game startup and before processing network physics data

### import_definition_structures
- Signature: `void import_definition_structures(void)`
- Purpose: Main entry point; orchestrates loading and importing physics definitions from disk
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes physics data, reads and unpacks WAD file, updates global physics state
- Calls: `init_physics_wad_data()`, `get_physics_wad_data()`, `import_physics_wad_data()`, `free_wad()`
- Notes: Invoked during game initialization; called once per game session

### get_network_physics_buffer
- Signature: `void *get_network_physics_buffer(long *physics_length)`
- Purpose: Serialize the current physics state for network transmission
- Inputs: `physics_length` ΓÇô pointer to store the serialized data size
- Outputs/Return: Pointer to flat serialized physics data (or NULL if no physics file set)
- Side effects: Allocates memory; `*physics_length` is set to the buffer size (or 0 on failure)
- Calls: `get_flat_data()`, `get_flat_data_length()`
- Notes: Receiver must free the returned buffer; data can be sent over network to other clients

### process_network_physics_model
- Signature: `void process_network_physics_model(void *data)`
- Purpose: Deserialize and apply physics data received from the network
- Inputs: `data` ΓÇô serialized physics buffer (may be NULL)
- Outputs/Return: None
- Side effects: Reinitializes physics data, inflates WAD data, unpacks and applies physics definitions
- Calls: `init_physics_wad_data()`, `inflate_flat_data()`, `import_physics_wad_data()`, `free_wad()`
- Notes: Safe to call with NULL; used during netgame join to sync physics state

### get_physics_wad_data (static)
- Signature: `static struct wad_data *get_physics_wad_data(bool *bungie_physics)`
- Purpose: Read and parse the physics WAD file from disk
- Inputs: `bungie_physics` ΓÇô pointer to bool to indicate if original Bungie physics were loaded
- Outputs/Return: Pointer to unpacked `wad_data` (or NULL on error); `*bungie_physics` indicates version
- Side effects: Opens/closes physics file; resets game errors on completion
- Calls: `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `set_game_error()`
- Notes: Checks for two valid physics data versions (`BUNGIE_PHYSICS_DATA_VERSION` or `PHYSICS_DATA_VERSION`); returns NULL if neither found

### import_physics_wad_data (static)
- Signature: `static void import_physics_wad_data(struct wad_data *wad)`
- Purpose: Extract and unpack all physics definition types from a WAD
- Inputs: `wad` ΓÇô parsed WAD data structure
- Outputs/Return: None
- Side effects: Unpacks and applies physics definitions to global game state; verifies data sizes
- Calls: `extract_type_from_wad()` (5 times for 5 physics tags), `unpack_monster_definition()`, `unpack_effect_definition()`, `unpack_projectile_definition()`, `unpack_physics_constants()`, `unpack_weapon_definition()`
- Notes: Uses assertions to verify that extracted data lengths match expected structure sizes and type counts; sets `PhysicsModelLoaded` flag if any data is found

## Control Flow Notes

**Initialization path:** Game startup calls `import_definition_structures()` ΓåÆ `init_physics_wad_data()` + `get_physics_wad_data()` + `import_physics_wad_data()`.

**Network path:** Client receives physics buffer ΓåÆ `process_network_physics_model()` ΓåÆ `init_physics_wad_data()` + `inflate_flat_data()` + `import_physics_wad_data()`.

The file-static functions (`get_physics_wad_data`, `import_physics_wad_data`) encapsulate the low-level WAD I/O and unpacking; the public functions provide high-level game-facing interfaces.

## External Dependencies

- **Includes (local):** `tags.h` (WAD tag constants), `map.h`, `interface.h`, `game_wad.h`, `wad.h` (WAD I/O), `game_errors.h`, `shell.h`, `preferences.h`, `FileHandler.h`, `monsters.h`, `effects.h`, `projectiles.h`, `player.h`, `weapons.h`, `physics_models.h`
- **Includes (external):** `cseries.h` (standard utilities), `<string.h>`
- **Symbols defined elsewhere:** `get_default_physics_spec()`, `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `free_wad()`, `extract_type_from_wad()`, `get_flat_data()`, `get_flat_data_length()`, `inflate_flat_data()`, `set_game_error()`, `unpack_*_definition()` functions (from physics modules), `init_*_definitions()` functions
