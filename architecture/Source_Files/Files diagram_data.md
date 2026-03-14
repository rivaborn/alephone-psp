# Source_Files/Files/AStream.cpp
## File Purpose
Implements type-safe binary serialization/deserialization with explicit Big Endian and Little Endian byte ordering. Provides input/output stream classes for network protocols and persistent storage, replacing less clear endianness handling from AlephOne's Packing system.

## Core Responsibilities
- Deserialization of 8/16/32-bit signed and unsigned integers with endianness control
- Serialization of integers to byte streams with specified byte order
- Raw byte reading/writing operations with bounds checking
- Stream state management (fail/bad bits) and exception-based error signaling
- Support for chaining operations via operator>> and operator<< return values

## External Dependencies
- `<string>` ΓÇö std::string for exception messages
- `<exception>` ΓÇö std::exception base class
- `<string.h>` ΓÇö memcpy() for bulk byte operations, strdup()/free() for message storage
- `"cstypes.h"` ΓÇö Type definitions (uint8, int8, uint16, int16, uint32, int32)
- Uses `std::` namespace

# Source_Files/Files/AStream.h
## File Purpose
Serialization/deserialization abstraction providing template-based binary stream I/O with explicit endianness handling. Replaces the less clear `Packing` classes from AlephOne with better type safety and clearer endian semantics. Supports bounds checking, exception handling, and state management similar to `std::iostream`.

## Core Responsibilities
- Define template-based stream base class (`basic_astream<T>`) for buffer-based I/O
- Provide abstract input stream (`AIStream`) with extraction operators (`operator>>`)
- Provide abstract output stream (`AOStream`) with insertion operators (`operator<<`)
- Implement big-endian and little-endian variants (`AIStreamBE`, `AIStreamLE`, `AOStreamBE`, `AOStreamLE`)
- Manage I/O state (good/bad/fail bits) and exception masking
- Enforce bounds checking and raise exceptions on buffer overflow
- Support reading/writing primitive types (int8, uint8, int16, uint16, int32, uint32) and raw byte arrays

## External Dependencies
- `#include <string>` ΓÇö for `std::string` in `failure` exception class.
- `#include <exception>` ΓÇö for `std::exception` base class.
- `#include "cstypes.h"` ΓÇö defines `int8`, `uint8`, `int16`, `uint16`, `int32`, `uint32`.

# Source_Files/Files/crc.cpp
## File Purpose
Implements CRC-32 and CRC-CCITT checksum generation for files and memory buffers in the Aleph One game engine. Provides table-based (lookup) CRC computation optimized for file integrity validation and data checksumming.

## Core Responsibilities
- Calculate CRC-32 checksums for unopened files (FileSpecifier) and opened file handles (OpenedFile)
- Calculate CRC-32 checksums for raw memory buffers
- Calculate CRC-CCITT checksums (non-reversed CCITT polynomial 0x1021)
- Manage dynamic allocation/deallocation of CRC-32 lookup table (lazy initialization)
- Handle chunked file I/O to support large files with fixed buffer size

## External Dependencies
- **FileHandler.h**: OpenedFile, FileSpecifier classes (file I/O abstraction)
- **cseries.h**: Type definitions (uint32, uint16, byte), platform macros
- Standard library: new/delete (stdlib.h), assert macro
- **Defined elsewhere:** OpenedFile::Open(), Close(), GetPosition(), SetPosition(), GetLength(), Read(); FileSpecifier::Open()

# Source_Files/Files/crc.h
## File Purpose
Header file declaring CRC (Cyclic Redundancy Check) calculation functions for the engine. Provides utilities to compute 32-bit and 16-bit checksums of files and raw data buffers for integrity validation.

## Core Responsibilities
- Declare 32-bit CRC calculation for files (by FileSpecifier and OpenedFile)
- Declare 32-bit CRC calculation for raw data buffers
- Declare 16-bit CCITT CRC calculation for data buffers
- Define interface between file I/O layer and CRC computation

## External Dependencies
- **Forward declarations:** `FileSpecifier`, `OpenedFile` (defined elsewhere, likely in file handling layer)
- **Primitive types:** `uint32`, `uint16`, `long` (defined in platform headers)
- **History note (Aug 15, 2000):** Refactored to use object-oriented file handler (FileSpecifier, OpenedFile)

# Source_Files/Files/extensions.h
## File Purpose
Header file declaring the physics data management interface for the Aleph One game engine (Marathon port). It provides functions to load physics files, import physics definitions, and synchronize physics models over the network.

## Core Responsibilities
- Set physics data source file
- Reset physics configuration to defaults
- Load and process physics data from file
- Serialize physics for network transmission
- Deserialize and apply network-received physics models

## External Dependencies
- `FileSpecifier` class (defined elsewhere; forward-declared)
- GNU GPL licensed; Bungie Studios / Aleph One project

# Source_Files/Files/FileHandler.h
## File Purpose
Provides cross-platform file and resource I/O abstractions for Marathon/Aleph One engine. Encapsulates Mac FSSpec/resource fork handling and SDL-based file access behind unified interfaces, supporting file creation, reading, writing, directory navigation, and resource management without exposing platform-specific details.

## Core Responsibilities
- Abstract opened file handles with position/length management and read/write operations
- Abstract loaded resources with automatic cleanup and Mac handle/SDL pointer management
- Abstract resource fork files with push/pop context stack for Mac or SDL equivalents
- Abstract file specification from paths (Unix-style on all platforms)
- Handle cross-platform directory navigation (Mac FSSpec IDs vs SDL paths)
- Support file type/typecode system and file creation with type attributes
- Provide file dialogs and disk operations (delete, copy, exchange, exists checks)
- List directory contents with metadata (SDL only)

## External Dependencies
- **<vector>, <string>** (STL): Dynamic collections for paths and directory listings
- **<SDL.h>** (SDL): `SDL_RWops` for cross-platform file I/O abstraction
- **Mac/Carbon APIs**: `FSSpec`, `OSErr`, `Handle`, `OSType` (Mac/Carbon.h)
- **tags.h**: `Typecode` enum, `FOUR_CHARS_TO_INT` macro, typecode access functions
- **<time.h>**: `time_t` / `TimeType` for file modification dates
- **Platform conditionals**: `#ifdef mac`, `#ifdef SDL`, `#ifdef __WIN32__` select implementations

# Source_Files/Files/FileHandler_SDL.cpp
## File Purpose
SDL-based implementation of platform-independent file handling for the Aleph One game engine. Provides abstractions for file I/O, resource management, file type detection, and modal dialogs for file selection, with transparent support for legacy Mac formats (AppleSingle, MacBinary).

## Core Responsibilities
- Wrap SDL_RWops file handles with error tracking and fork/offset handling for Mac compatibility
- Load and manage game resources (maps, sounds, shapes, physics data)
- Detect file types by reading magic bytes and version headers
- Search data directories for files using configurable search paths
- Provide directory traversal and file listing with custom widgets
- Implement modal file selection dialogs (open/save) with directory browsing
- Canonicalize and normalize file paths across platforms (Windows, Mac, Unix)
- Copy, rename, exchange, and delete files with error reporting

## External Dependencies
- **SDL**: SDL_RWops, SDL_RWFromFile, SDL_RWread/write/seek/tell/close; SDL endian macros (SDL_ReadBE32, SDL_ReadBE16)
- **System**: stdio.h (FILE in non-Mac paths), stdlib.h, errno.h, limits.h, string, vector
- **Platform-specific**: 
  - Windows: windows.h, direct.h (mkdir), io.h (access), sys/stat.h (stat)
  - Unix: sys/stat.h, fcntl.h, dirent.h, unistd.h
- **Aleph One engine**: cseries.h, FileHandler.h, resource_manager.h (open_res_file, close_res_file, use_res_file, has_1_resource, get_1_resource), shell.h (search paths), interface.h (get_game_state, update_game_window), game_errors.h (set_game_error), tags.h (typecodes, FOUR_CHARS_TO_INT, tags like LINE_TAG, MONSTER_PHYSICS_TAG), sdl_dialogs.h (dialog, widget, placer classes), sdl_widgets.h (w_title, w_spacer, w_button, w_static_text, w_text_entry), SoundManager.h (play_dialog_sound)
- **Mac-specific**: mac_rwops.h (open_fork_from_existing_path), Functions for AppleSingle/MacBinary detection (is_applesingle, is_macbinary, extern from unknown source)

# Source_Files/Files/filetypes_macintosh.cpp
## File Purpose
Manages macOS file typecodes for the Aleph One game engine, mapping between OSType (macOS four-character file type codes) and internal Typecode enums. Loads custom typecodes from the resource fork and maintains backwards compatibility with Marathon 2 file types.

## Core Responsibilities
- Load custom FTyp resource from macOS resource fork (resource 128)
- Maintain bidirectional mapping between OSType and Typecode
- Support multiple OSType mappings to single internal Typecode (M2 compatibility)
- Provide accessors to query and modify typecode mappings
- Initialize runtime lookup map on startup
- Handle boundary conditions and unsupported typecodes gracefully

## External Dependencies
- **macOS API:** GetResource, GetHandleSize, HLock, HUnlock, ReleaseResource (Carbon/resource management)
- **Standard library:** std::map, std::vector, string.h
- **Engine headers:** tags.h (Typecode enum definition), csalerts.h (assert/alert macros)
- **Conditional:** Carbon/Carbon.h (if EXPLICIT_CARBON_HEADER defined)

# Source_Files/Files/find_files.h
## File Purpose
Header file defining cross-platform file-finding abstractions for macOS (classic Mac API) and SDL-based platforms. Enables searching for files by typecode with support for recursive directory traversal, filtering via callbacks, and directory-change notifications.

## Core Responsibilities
- Define platform-specific `FileFinder` class hierarchy for file discovery
- Support file enumeration by typecode with optional recursion
- Provide callback mechanisms for filtering and processing found files
- Abstract differences between macOS Classic/Carbon APIs and SDL file operations
- Define `WILDCARD_TYPE` constant for unrestricted type matching

## External Dependencies
- **Includes**: `FileHandler.h` (provides `FileSpecifier`, `DirectorySpecifier`, `Typecode`, `OpenedFile`, `OpenedResourceFile`, `tags.h`)
- **macOS platform**: `Carbon/Carbon.h` (or `Files.h`, `Resources.h`); uses `CInfoPBRec`, `OSType`, `OSErr`, `DirectorySpecifier`
- **SDL platform**: `<vector>`, `<string>`, `SDL.h`; uses `vector<FileSpecifier>`, `std::string`
- **Symbols defined elsewhere**: `FileSpecifier`, `DirectorySpecifier`, `Typecode`, `_typecode_unknown`, `OpenedFile`, `OpenedResourceFile`

# Source_Files/Files/find_files_sdl.cpp
## File Purpose
SDL implementation of recursive file searching for the Aleph One game engine. Provides a template-method pattern where `FileFinder::Find()` handles directory traversal and type filtering, while derived classes override `found()` to process matching files.

## Core Responsibilities
- Recursively traverse directories to locate files matching a given type code
- Sort directory entries (directories before files, then alphabetically)
- Filter files by type code or accept wildcard matches
- Support early termination on first match or continue-all-files collection via virtual callback
- Construct full file paths during traversal using `FileSpecifier` operators

## External Dependencies
- **Includes:** `cseries.h` (STL, types), `FileHandler.h` (`FileSpecifier`, `DirectorySpecifier`, `dir_entry`), `find_files.h` (class declarations)
- **Uses:** `std::vector`, `std::sort`, `std::vector::const_iterator`
- **Defined elsewhere:** `DirectorySpecifier::ReadDirectory()`, `FileSpecifier::GetType()`, `WILDCARD_TYPE` constant

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

## External Dependencies
- **Notable includes:** map.h, monsters.h, network.h, projectiles.h, effects.h, player.h, platforms.h, wad.h, FileHandler.h, Packing.h, computer_interface.h, XML_LevelScript.h, ChaseCam.h, Music.h, SoundManager.h.
- **Defined elsewhere (partial list):**
  - Core structures: `wad_data`, `wad_header`, `game_data`, `dynamic_world`, `static_world`, `entry_point`, `map_object`, `static_data`, etc. (map.h, wad.h)
  - Globals: `objects`, `map_endpoints`, `map_lines`, `map_sides`, `map_polygons`, `monsters`, `effects`, `projectiles`, `platforms`, `saved_objects`, `map_indexes`, `players`, `dynamic_world`, `automap_lines`, `automap_polygons`
  - Functions: `open_wad_file_for_reading()`, `read_wad_header()`, `extract_type_from_wad()`, `append_data_to_wad()`, `create_empty_wad()`, `inflate_flat_data()`, `get_flat_data()`, `free_wad()`, many pack/unpack functions (Packing.h)
  - Game callbacks: `entering_map()`, `leaving_map()`, `goto_level()`, `RunLevelScript()`, `RunRestorationScript()`, `initialize_map_for_new_game()`, `new_player()`, `new_monster()`, `entering_map()`, `reset_motion_sensor()`, `ChaseCam_Initialize()`

# Source_Files/Files/game_wad.h
## File Purpose
Header declaring the game's save/load system and WAD (map data) management. Provides functions to persist game state to disk, manage save files and map checksums, and control pause/resume behavior. Interfaces with the engine's file handling subsystem.

## Core Responsibilities
- Save game state and export levels to disk
- Manage current saved game file specification
- Process and validate map WAD data structures
- Load maps and verify integrity via checksums
- Switch between map files
- Pause and resume game execution
- Retrieve save game file descriptors

## External Dependencies
- `FileSpecifier` (defined elsewhere; forward declared here)
- `wad_data` struct (definition not in this file)
- File I/O subsystem (implementation hidden)

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

## External Dependencies

- **Includes (local):** `tags.h` (WAD tag constants), `map.h`, `interface.h`, `game_wad.h`, `wad.h` (WAD I/O), `game_errors.h`, `shell.h`, `preferences.h`, `FileHandler.h`, `monsters.h`, `effects.h`, `projectiles.h`, `player.h`, `weapons.h`, `physics_models.h`
- **Includes (external):** `cseries.h` (standard utilities), `<string.h>`
- **Symbols defined elsewhere:** `get_default_physics_spec()`, `open_wad_file_for_reading()`, `read_wad_header()`, `read_indexed_wad_from_file()`, `close_wad_file()`, `free_wad()`, `extract_type_from_wad()`, `get_flat_data()`, `get_flat_data_length()`, `inflate_flat_data()`, `set_game_error()`, `unpack_*_definition()` functions (from physics modules), `init_*_definitions()` functions

# Source_Files/Files/Packing.cpp
## File Purpose
Implements byte-order conversion functions for serializing/deserializing game data. Converts between native in-memory values (int16, uint16, int32, uint32) and byte streams in either big-endian or little-endian format. Core infrastructure for Marathon's packed file format handling.

## Core Responsibilities
- Read values from byte streams in big-endian format (StreamToValueBE)
- Read values from byte streams in little-endian format (StreamToValueLE)
- Write values to byte streams in big-endian format (ValueToStreamBE)
- Write values to byte streams in little-endian format (ValueToStreamLE)
- Support signed and unsigned 16-bit and 32-bit integers
- Automatically advance stream pointers during read/write operations
- Handle sign-extension when converting from unsigned to signed representations

## External Dependencies
- `cseries.h`: Provides base type definitions (uint8, int16, uint16, int32, uint32)
- `Packing.h`: Function declarations and macro configuration
- Implicit: SDL.h, SDL_byteorder.h (included transitively via cseries.h)

# Source_Files/Files/Packing.h
## File Purpose
Provides utility functions for serializing and deserializing numerical values and raw byte data between native memory layouts and packed big-endian or little-endian byte streams. Used throughout the Marathon series game engine to handle cross-platform data format conversion without relying on compiler-generated padding.

## Core Responsibilities
- Serialize/deserialize single numerical values (16-bit and 32-bit integers, signed and unsigned) to/from byte streams
- Serialize/deserialize arrays of numerical values maintaining stream pointer advancement
- Copy arbitrary byte blocks to/from streams with stream pointer management
- Abstract endianness conversion (big-endian vs. little-endian) via preprocessor macros
- Provide a consistent API where packing and unpacking routines are syntactically similar

## External Dependencies
- `memcpy` (from `<string.h>`, included but commented out; likely included elsewhere)
- Built-in integer types: `uint8`, `int16`, `uint16`, `int32`, `uint32`
- Actual implementations in `Packing.cpp` (moved there to avoid compiler inlining issues)

# Source_Files/Files/preprocess_map_sdl.cpp
## File Purpose
SDL implementation of save game file handling and default data file localization. Provides functions to locate critical game data files (maps, shapes, sounds, physics models) from a configurable search path, and implements save/load game dialog functionality.

## Core Responsibilities
- Locate default game data files (map, physics model, shapes, sounds, music, theme) from a search path
- Display read/write dialogs for save game selection and creation
- Handle game state persistence (save_game and related game_wad functions)
- Alert user when critical files (map, shapes) cannot be found
- Support graceful degradation when optional files (physics, sounds) are missing

## External Dependencies
- **cseries.h** ΓÇö Core types, macros, platform abstractions (vector, string)
- **FileHandler.h** ΓÇö `FileSpecifier` and `DirectorySpecifier` classes for cross-platform file operations
- **world.h, map.h** ΓÇö Game world/map data structures (not directly used; for context)
- **shell.h** ΓÇö Global functions (`pause_game`, `resume_game`, `show_cursor`, `hide_cursor`); `strFILENAMES` resource constants
- **interface.h** ΓÇö Game interface enums (`strFILENAMES` filenames, `strERRORS` error codes, `strPROMPTS` prompt strings)
- **game_wad.h** ΓÇö `save_game_file()`, `get_current_saved_game_name()` (defined elsewhere)
- **game_errors.h** ΓÇö Error type constants
- **shell_sdl.cpp (external)** ΓÇö `data_search_path` vector (defined elsewhere, used here)
- **STL vector** ΓÇö Dynamic array for search path iteration

# Source_Files/Files/preprocess_map_shared.cpp
## File Purpose

Implements automatic game-save functionality for the Aleph One engine (Marathon-like game). Provides mechanisms to save games without user interaction, generate filename-friendly strings, and automatically resolve filename conflicts by appending numeric suffixes.

## Core Responsibilities

- Auto-save game state without presenting save dialogs to the user
- Sanitize arbitrary text into filename-safe strings (alphanumeric + spaces only)
- Generate non-conflicting filename variants by appending numeric suffixes (e.g., "Map", "Map 2", "Map 3")
- Report save success/failure to screen via `screen_printf()`
- Support overwriting recent saves or creating new unique filenames based on current level name

## External Dependencies

- **FileHandler.h**: `FileSpecifier`, `DirectorySpecifier` classes for filesystem abstraction
- **game_wad.h**: `get_current_saved_game_name()`, `save_game_file()` (save/load mechanisms)
- **interface.h**: `TS_GetCString()` (string resource lookup), `screen_printf()` (on-screen messaging)
- **map.h**: `static_world->level_name` (current level name from world state)
- **shell.h**: `screen_printf()` (display function)
- **TextStrings.h**: `TS_GetCString()` (localized string fetching)
- **Standard C**: `<ctype.h>` (`isalnum()`), `<string.h>` implicit (`strcpy()`, `strlen()`, `memset()`)

# Source_Files/Files/resource_manager.cpp
## File Purpose
Provides cross-platform abstraction for MacOS resource fork files, handling AppleSingle, MacBinary II, and raw resource fork formats. Maintains a stack of open resource files with format-transparent parsing and a query API for retrieving resources by type, ID, or index.

## Core Responsibilities
- Detect and transparently handle AppleSingle, MacBinary II, and raw resource fork formats
- Parse resource maps from files to build in-memory type/ID lookup tables
- Manage a stack of open resource files with a "current" file pointer
- Provide query APIs to count, enumerate, and load resources by type/ID or index
- Handle file I/O via SDL_RWops abstraction for cross-platform compatibility
- Support fallback file naming conventions (`.rsrc`, `.resources`, `/..namedfork/rsrc`)

## External Dependencies
- **SDL_RWops** (SDL_rwops.h) ΓÇô file abstraction; endian read functions (SDL_ReadBE32, etc.)
- **FileHandler.h** ΓÇô FileSpecifier, LoadedResource, platform-specific file operations
- **Logging.h** ΓÇô logNote, logTrace, logAnomaly, logDump4 macros
- **cseries.h** ΓÇô uint32, uint16, uint8 typedef; platform macros (__MACOS__, __BEOS__)
- **csfiles_beos.cpp** (defined elsewhere) ΓÇô `has_rfork_attribute()`, `sdl_rw_from_rfork()`
- **mac_rwops.h** (macOS only) ΓÇô `open_fork_from_existing_path()`

# Source_Files/Files/resource_manager.h
## File Purpose
Header file providing cross-platform resource file management abstraction for the Aleph One engine (a Marathon port). Wraps SDL_RWops to emulate MacOS Classic resource forking on non-Mac platforms, enabling asset loading from structured resource containers.

## Core Responsibilities
- Initialize and manage resource file contexts
- Open/close resource files and track the current active file
- Count and enumerate resource IDs by type code
- Retrieve resources by ID or index
- Check resource existence
- Provide both "1" (single file) and default (search hierarchy) variants of operations

## External Dependencies
- `stdio.h` ΓÇô Basic I/O (likely for legacy reasons; SDL provides RWops)
- `<vector>` ΓÇô Dynamic ID/resource lists
- `<SDL.h>` ΓÇô RWops file abstraction
- **Forward declared:** FileSpecifier, LoadedResource (defined in separate headers)
- **uint32** ΓÇô Assumed from stdint.h or engine-specific typedef

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

## External Dependencies
- **cstypes.h**: `uint32` type, `FOUR_CHARS_TO_INT(a,b,c,d)` macro, `NONE` constant
- **\<vector\>**: C++ standard library (used in Mac-specific function returning `std::vector<OSType>`)
- **Platform-specific includes** (cstypes.h handles): Carbon.h (Mac), SDL_types.h (SDL), SupportDefs.h (BeOS)

# Source_Files/Files/wad.cpp
## File Purpose
Implements WAD file I/O and data management for the Aleph One game engine (Marathon). WAD files are container formats holding game assets (maps, textures, sprites, physics data). This file provides APIs to read/write WAD files from disk, convert between binary and in-memory representations, manage game data, verify file integrity via checksums, and handle multiple WAD file versions for backward compatibility.

## Core Responsibilities
- **Read/Write WAD files**: Load WAD headers and indexed data blocks from disk; write structured WAD data back to files with proper formatting
- **Version compatibility**: Support multiple WAD file versions (pre-entry-point, directory-entry, overlays, Infinity) with appropriate structure sizing and validation
- **Memory management**: Allocate and free WAD data structures; support both read-only (pointer-based) and modifiable (copied) loaded WADs
- **Data extraction & modification**: Query WAD contents by tag; append/remove/replace data entries; manage tag arrays dynamically
- **Binary serialization**: Pack/unpack data structures to/from byte streams with endianness conversion (big-endian file format vs. native machine byte order)
- **Integrity checking**: Calculate and verify CRC32 checksums for file validation; store parent/child checksums for patch file relationships
- **Network serialization**: Flatten WAD data for transmission with encapsulated headers; reconstruct from received binary blobs
- **File I/O abstraction**: Wrap platform-specific file operations (OpenedFile class) and handle seek/read/write positioning

## External Dependencies
- **cseries.h** ΓÇô Base types, macros, utilities (e.g., `obj_clear()`, `objlist_copy()`, memory functions)
- **tags.h** ΓÇô Tag type constants (e.g., `POINT_TAG`, `MAP_INFO_TAG`, `PLAYER_STRUCTURE_TAG`), typecode enums
- **crc.h** ΓÇô `calculate_crc_for_opened_file()` for file integrity checking
- **game_errors.h** ΓÇô `set_game_error()` for error reporting
- **interface.h** ΓÇô `alert_user()`, error string resources (strERRORS), UI dialogs
- **FileHandler.h** ΓÇô `OpenedFile`, `FileSpecifier` classes (file I/O abstraction)
- **Packing.h** ΓÇô `StreamToValue()`, `ValueToStream()`, `StreamToBytes()`, `BytesToStream()` (endian conversion macros)
- **stdlib.h, string.h** ΓÇô Standard C library (malloc, free, memcpy, strlen)

**Defined elsewhere:**
- `level_transition_malloc()` ΓÇô Custom allocator (defined in Marathon-specific code; see comment)
- `free_wad()` ΓÇô Memory deallocation (header declares it; implementation not in this file)
- `temporary` ΓÇô Global char buffer for debug messages

# Source_Files/Files/wad.h
## File Purpose
Header file defining the WAD (game resource file) format and API for the Marathon game engine. WAD files are binary containers that hold game data (maps, sprites, sounds, physics models, etc.) organized as tagged data structures. Supports versioning, checksums, and multiple WAD files per container for modular content.

## Core Responsibilities
- Define binary WAD file format (headers, directory structures, entry headers) with version compatibility
- Declare file I/O operations (create, open, close, read, write WAD files and headers)
- Provide data extraction and manipulation API (append/remove tagged data, flatten/inflate WAD structures)
- Support checksum validation and parent file references for data patches
- Manage in-memory WAD representation (tag collections with offset tracking)
- Provide "between levels" state management for memory allocation control

## External Dependencies
- **Included**: `tags.h` ΓÇö defines `Typecode` enum and tag constants (`POINT_TAG`, `POLYGON_TAG`, `MAP_INFO_TAG`, etc.)
- **Defined elsewhere**: 
  - `FileSpecifier` class ΓÇö file path abstraction layer
  - `OpenedFile` class ΓÇö file handle/stream abstraction
  - Standard C types (`uint32`, `int32`, `int16`, `byte`, `long`)
  - `FOUR_CHARS_TO_INT` macro (likely from `tags.h` or `cstypes.h`)

# Source_Files/Files/wad_prefs.cpp
## File Purpose
Manages persistent player preferences storage for the Aleph One game engine using WAD (Where's All the Data) file format. Handles initialization, loading, retrieval with lazy initialization/validation, and saving of preferences with automatic error recovery for corrupted files.

## Core Responsibilities
- Initialize and open preferences file (create if missing, delete and recreate if corrupted)
- Load preference WAD from disk into memory
- Retrieve individual preference entries by tag with lazy initialization and optional validation
- Write modified preferences back to disk in WAD format
- Provide callback hooks for preference initialization and validation
- Handle platform-specific file paths (Mac vs. SDL)
- Implement graceful error recovery and reporting

## External Dependencies
- **Includes:** cseries.h, wad.h, game_errors.h, wad_prefs.h, FileHandler.h, string.h, stdlib.h, stdexcept
- **External symbols:** `create_empty_wad()`, `extract_type_from_wad()`, `append_data_to_wad()`, `free_wad()` (WAD ops); `open_wad_file_for_reading/writing()`, `read_wad_header()`, `write_wad_header()`, `write_directorys()`, `calculate_wad_length()` (file I/O); `set_game_error()`, `error_pending()` (error handling); `MemError()` (Mac); `dprintf()` (debug)

# Source_Files/Files/wad_prefs.h
## File Purpose
Provides interfaces for managing WAD (game data) preference files, including opening, reading, validating, and writing preferences to persistent storage. Supports both programmatic access and macOS-specific dialog-based preference UI configuration.

## Core Responsibilities
- Define function signatures for preference file I/O (open, read, write)
- Declare callback function pointer types for preference initialization and validation
- Manage internal preference data storage and file references
- Provide macOS-specific dialog UI structure for interactive preference configuration

## External Dependencies
- **FileHandler.h**: `FileSpecifier` (file path abstraction), `OpenedResourceFile` (resource I/O)
- **Tags.h** (via FileHandler.h): `Typecode` enum for file types
- **Undefined in this file**: `WadDataType`, `struct wad_data`, `w_get_data_from_preferences()` implementation

# Source_Files/Files/wad_sdl.cpp
## File Purpose
Provides SDL-based file discovery functions for locating map files (WAD files) by checksum or modification date within a configurable search path. Implements searchable catalog functionality for the Aleph One game engine's resource loading system.

## Core Responsibilities
- Search for map files across multiple directories by checksum validation
- Locate files by modification date across the search path
- Abstract checksum and date matching logic via FileFinder subclasses
- Iterate through `data_search_path` directories to find resource files
- Read and parse WAD file checksums at fixed offset (0x44)

## External Dependencies
- `#include "cseries.h"` ΓÇö common type definitions and platform macros
- `#include "FileHandler.h"` ΓÇö `FileSpecifier`, `OpenedFile`, `DirectorySpecifier`, `TimeType` classes
- `#include "find_files.h"` ΓÇö `FileFinder` base class, `Typecode` type
- `#include <SDL_endian.h>` ΓÇö `SDL_ReadBE32()` for big-endian checksum reads
- **Defined elsewhere**: `data_search_path` (extern vector from shell_sdl.cpp)


