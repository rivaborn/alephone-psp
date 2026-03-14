# Source_Files/Files/extensions.h

## File Purpose
Header file declaring the physics data management interface for the Aleph One game engine (Marathon port). It provides functions to load physics files, import physics definitions, and synchronize physics models over the network.

## Core Responsibilities
- Set physics data source file
- Reset physics configuration to defaults
- Load and process physics data from file
- Serialize physics for network transmission
- Deserialize and apply network-received physics models

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (forward decl.) | File path/specification handler for physics data files |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| BUNGIE_PHYSICS_DATA_VERSION | #define | static | Legacy physics format version identifier (0) |
| PHYSICS_DATA_VERSION | #define | static | Current physics format version identifier (1) |

## Key Functions / Methods

### set_physics_file
- Signature: `void set_physics_file(FileSpecifier& File);`
- Purpose: Designate the physics data file to load
- Inputs: FileSpecifier reference (file path/identifier)
- Outputs: None
- Side effects: Changes active physics file source
- Notes: Does not immediately load; paired with `import_definition_structures()`

### set_to_default_physics_file
- Signature: `void set_to_default_physics_file(void);`
- Purpose: Reset physics file to engine default
- Inputs: None
- Outputs: None
- Side effects: Resets active physics to built-in default

### import_definition_structures
- Signature: `void import_definition_structures(void);`
- Purpose: Parse and load all physics data from current physics file
- Inputs: None (uses previously set physics file)
- Outputs: None
- Side effects: Allocates/loads physics models into game state
- Notes: Likely called during level or game initialization

### get_network_physics_buffer / process_network_physics_model
- Signatures: `void *get_network_physics_buffer(long *physics_length);` / `void process_network_physics_model(void *data);`
- Purpose: Serialize physics for multiplayer sync; deserialize received physics
- Inputs/Outputs: Raw buffer pointers; length written/read from caller
- Side effects: `get_*` allocates/returns buffer; `process_*` modifies game physics state
- Notes: Raw void pointers suggest byte-level serialization format; version constants likely guard compatibility

## Control Flow Notes
Typical initialization flow: `set_physics_file()` ΓåÆ `import_definition_structures()`. Network flow: `get_network_physics_buffer()` (send) Γåö `process_network_physics_model()` (receive).

## External Dependencies
- `FileSpecifier` class (defined elsewhere; forward-declared)
- GNU GPL licensed; Bungie Studios / Aleph One project
