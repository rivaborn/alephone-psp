# Subsystem Overview

## Purpose
The Files subsystem provides cross-platform file I/O, resource management, and data serialization for the Aleph One game engine (Marathon port). It abstracts platform-specific file handling (Mac FSSpec/resource forks, SDL, Windows) and implements binary serialization with explicit endianness control, WAD file format processing, game save/load persistence, and file discovery mechanisms.

## Key Files
| File | Role |
|------|------|
| FileHandler.h / FileHandler_SDL.cpp | Cross-platform file and resource abstraction (FileSpecifier, OpenedFile, DirectorySpecifier, LoadedResource) |
| AStream.h / AStream.cpp | Type-safe binary serialization/deserialization with big-endian and little-endian support; exception-based error handling |
| Packing.h / Packing.cpp | Byte-order conversion functions (StreamToValueBE/LE, ValueToStreamBE/LE) for 16/32-bit integers |
| tags.h | Game data structure identifiers (Typecode enum, 4-character WAD tags); platform-agnostic file type mappings |
| wad.h / wad.cpp | WAD file format I/O and in-memory management; versioning, checksums, tagged data extraction/modification |
| game_wad.h / game_wad.cpp | Game save/load system and map persistence; coordinates level initialization and state serialization |
| find_files.h / find_files_sdl.cpp | File searching with type-code filtering and recursive directory traversal |
| resource_manager.h / resource_manager.cpp | Cross-platform resource fork handling (AppleSingle, MacBinary II, raw format); file context stack |
| crc.h / crc.cpp | CRC-32 and CRC-CCITT checksum generation for files and buffers |
| wad_prefs.h / wad_prefs.cpp | Persistent preferences storage using WAD format with lazy initialization and validation |
| preprocess_map_sdl.cpp | Save/load dialogs, default game file localization from search paths |
| preprocess_map_shared.cpp | Auto-save functionality with filename conflict resolution |
| import_definitions.cpp | Physics model loading and network synchronization (monsters, effects, weapons) |
| extensions.h | Physics data management interface |
| filetypes_macintosh.cpp | macOS file typecode mappings (OSType Γåö Typecode) |
| wad_sdl.cpp | WAD file discovery by checksum or modification date |

## Core Responsibilities
- Abstract cross-platform file I/O primitives (FileSpecifier, OpenedFile) for Windows, Mac, Unix/Linux, and PSP
- Provide unified resource fork handling for macOS-compatible formats (AppleSingle, MacBinary II) on non-Mac platforms
- Implement binary serialization with explicit endianness control (big-endian file format vs. native byte order)
- Parse, create, and modify WAD files (tagged binary data containers for game assets); support versioning and checksums
- Manage game state persistence (save/load games, map data, player state, world objects)
- Provide file discovery mechanisms (search by typecode, checksum, modification date; recursive directory traversal)
- Calculate and verify CRC-32 checksums for file integrity validation and patch file relationships
- Load and synchronize physics definitions (monsters, effects, weapons, projectiles) from disk and network
- Maintain persistent player preferences in WAD format with error recovery
- Abstract platform-specific file type systems (macOS FSSpec typecodes, Unix magic bytes)

## Key Interfaces & Data Flow
**Exposes to other subsystems:**
- FileHandler abstractions (FileSpecifier, OpenedFile, DirectorySpecifier, LoadedResource, OpenedResourceFile)
- Binary stream classes (AStream, AIStream/AOStream for endian-aware serialization)
- Packing utility functions (StreamToValueBE/LE, ValueToStreamBE/LE)
- WAD APIs (open_wad_file_for_reading/writing, append_data_to_wad, extract_type_from_wad, flatten/inflate)
- Game persistence (save_game_file, get_current_saved_game_name, use_map_file)
- Physics loading (load_physics_file, import_physics_definitions)
- File discovery (FileFinder, file searching by checksum/date/type)
- Checksum functions (calculate_crc_for_opened_file, calculate_crc_for_buffer)

**Consumes from other subsystems:**
- GameWorld/RenderMain: game state structures (player, monsters, objects, platforms, effects) for serialization
- Network subsystem: physics data transmission format and protocol
- Misc/Tags: Typecode enum, tag constants (POINT_TAG, MONSTER_PHYSICS_TAG, etc.)
- Misc/GameErrors: error reporting (set_game_error, error_pending)
- Sound/Audio: sound resource loading via resource manager
- Preferences/XML: preference structure definitions
- Platform layer: SDL_RWops primitives (for PSP: PSP SDK file functions via SDL)

## Runtime Role
**Initialization (startup/level entry):**
- Open and parse preferences WAD file; initialize with defaults if missing or corrupted
- Locate default game data files (map, physics, shapes, sounds) via configurable search paths
- Initialize resource manager stack and load resource file contexts
- Load physics definitions from disk-based WAD files for current game session

**During gameplay (frame-by-frame):**
- Not part of render/game loop; file operations are on-demand only
- Respond to dynamic resource requests (load additional WAD data as needed)
- Lazy-load physics definitions over network in multiplayer games

**Shutdown/level exit:**
- Serialize game state (player position, monsters, effects, platform states) to save WAD
- Flush preferences WAD to disk with updated player settings
- Close open resource files and free WAD data structures
- Release CRC-32 lookup table if allocated

## Notable Implementation Details
- All file I/O routed through SDL_RWops abstraction for cross-platform uniformity; PSP porting uses PSPSDK file wrappers via SDL
- WAD format uses big-endian byte order for file storage; Packing layer converts to/from native byte order at serialization boundaries
- CRC-32 lookup table allocated on first use (lazy initialization) and retained for engine lifetime
- Resource manager maintains a stack of open resource file contexts with push/pop semantics; supports simultaneous access to multiple resource files
- File type system decoupled from platform via Typecode enum; macOS resource fork mappings loaded from Carbon resource 128; SDL/Windows platforms use magic-byte detection
- Physics data supports bidirectional serialization: diskΓåÆmemory (game load) and memoryΓåÆnetwork (multiplayer sync)
- AStream provides exception-based error signaling; fails on buffer overflow with explicit failure bits (analogous to std::iostream)
- Preferences WAD supports optional validation callbacks; corrupted files trigger automatic deletion and recreation with defaults
</output>
