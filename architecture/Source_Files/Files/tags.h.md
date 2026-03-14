# Source_Files/Files/tags.h

## File Purpose
Defines game data structure identifiers (tags) and file type codes for the Marathon/Aleph One engine. Provides a platform-agnostic typecode system to map game-level file types (scenario, savegame, physics, etc.) to platform-specific file type constants, and defines 4-character tags for all serialized game data structures.

## Core Responsibilities
- Define `Typecode` enum abstracting platform-specific file type details
- Initialize and manage typecode mappings (load from resource fork on Mac)
- Provide accessor functions for typecode values
- Define constant tags (4-char codes via `FOUR_CHARS_TO_INT`) for:
  - Level map data (points, lines, sides, polygons, light sources, annotations, etc.)
  - Game objects and AI (items, monsters, platforms, doors, terminals, etc.)
  - Save/load state (player, monsters, weapons, effects, projectiles, etc.)
  - Physics and shapes subsystems
  - Preferences (graphics, network, input, sound, etc.)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Typecode` | enum | Abstract file type identifier; values mapped to platform-specific OS types |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_typecodes
- Signature: `void initialize_typecodes(void)`
- Purpose: Load typecode mappings from the platform resource fork (Mac) or equivalent
- Inputs: None
- Outputs/Return: None (populates internal storage)
- Side effects: Initializes global/static typecode table
- Notes: Required before calling `get_typecode()`; implementation in filetypes_macintosh.c

### get_typecode
- Signature: `uint32 get_typecode(Typecode which)`
- Purpose: Retrieve the platform-specific file type code for an abstract typecode
- Inputs: `which` ΓÇô typecode enum value
- Outputs/Return: Platform-specific type code (uint32); defaults to `'????' ` for unknown types
- Side effects: None
- Calls: Queries internal mapping table
- Notes: Replaced compile-time `#define` macros to support runtime platform abstraction

### set_typecode
- Signature: `void set_typecode(Typecode which, uint32 _type)`
- Purpose: Override a typecode mapping (e.g., during initialization or testing)
- Inputs: `which` ΓÇô typecode enum; `_type` ΓÇô new platform-specific code
- Outputs/Return: None
- Side effects: Modifies global typecode table
- Calls: Direct table update

### get_typecode_for_file_type (Mac only)
- Signature: `Typecode get_typecode_for_file_type(OSType inType)`
- Purpose: Reverse lookupΓÇöfind abstract typecode from platform-specific OS type
- Inputs: `inType` ΓÇô MacOS OSType
- Outputs/Return: Matching `Typecode` enum value
- Notes: Conditional on `#ifdef mac`

### get_all_file_types_for_typecode (Mac only)
- Signature: `const std::vector<OSType> get_all_file_types_for_typecode(Typecode which)`
- Purpose: Retrieve all MacOS file types associated with a single abstract typecode (e.g., multiple extensions)
- Inputs: `which` ΓÇô typecode enum
- Outputs/Return: Vector of OSType values
- Notes: Conditional on `#ifdef mac`; uses C++ vector

## Control Flow Notes
This header is used during resource loading and game serialization:
- **Initialization**: `initialize_typecodes()` called early in engine startup (loads Mac resource fork typecodes)
- **File I/O**: `get_typecode()` called when opening/saving scenario files, savegames, physics data, etc.
- **Tag usage**: Tag constants (`POINT_TAG`, `POLYGON_TAG`, etc.) are used as keys when reading/writing level and save data structures from WAD/archive files
- **Mac-only**: Reverse lookup functions enable file dialogs and type validation on MacOS

## External Dependencies
- **cstypes.h**: `uint32` type, `FOUR_CHARS_TO_INT(a,b,c,d)` macro, `NONE` constant
- **\<vector\>**: C++ standard library (used in Mac-specific function returning `std::vector<OSType>`)
- **Platform-specific includes** (cstypes.h handles): Carbon.h (Mac), SDL_types.h (SDL), SupportDefs.h (BeOS)
