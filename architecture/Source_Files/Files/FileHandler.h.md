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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| OpenedFile | class | Open file handle with seek/read/write/length operations |
| LoadedResource | class | Manages resource data lifetime with auto-unload on destruction |
| OpenedResourceFile | class | Abstracts Mac resource forks or SDL file resource access |
| DirectorySpecifier | class (Mac) / typedef (SDL) | Directory location via vRefNum/parID (Mac) or path (SDL) |
| dir_entry | struct | Directory listing entry with name, size, flags (SDL) |
| FileSpecifier | class | File specification and operations (main API surface) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| RefNum_Closed | const short | global | Sentinel value for closed Mac file reference (-1) |
| isCurrentRefRWOps | static bool | OpenedResourceFile class | Tracks if current resource ref is SDL RWops vs Mac |

## Key Functions / Methods

### FileSpecifier::SetNameWithPath
- Signature: `bool SetNameWithPath(const char *NameWithPath)`
- Purpose: Parse and set file path; locate file in data directories (SDL) or relative to root (Mac)
- Inputs: Unix-style path (e.g., "maps/level01.scn")
- Outputs/Return: true if found/valid, false otherwise
- Side effects: Updates internal filespec
- Calls: None visible
- Notes: Translates ':' to '/' on Mac; SDL searches all registered data directories

### FileSpecifier::Open
- Signature: `bool Open(OpenedFile& OFile, bool Writable=false)` or `bool Open(OpenedResourceFile& OFile, bool Writable=false)`
- Purpose: Open file for standard I/O (first overload) or resource access (second overload)
- Inputs: OFile (output), Writable (optional, default false)
- Outputs/Return: true if opened, OFile populated with handle
- Side effects: Opens file/resource fork on disk
- Calls: None visible
- Notes: OFile auto-closes on destruction

### OpenedResourceFile::Get
- Signature: `bool Get(uint32 Type, int16 ID, LoadedResource& Rsrc)`
- Purpose: Load resource by type and ID into managed container
- Inputs: Type (4-char typecode), ID (resource identifier), Rsrc (output)
- Outputs/Return: true if found and loaded, Rsrc populated
- Side effects: Allocates/acquires resource data
- Calls: None visible
- Notes: Overloaded with char parameter variants; requires Mac push/pop on that platform

### FileSpecifier::GetType
- Signature: `Typecode GetType()`
- Purpose: Identify file type using tags.h typecode system
- Inputs: None
- Outputs/Return: Typecode enum value (_typecode_unknown if unidentified)
- Side effects: None
- Calls: None visible
- Notes: Uses platform type attributes or suffix conventions

### FileSpecifier::ReadDirectory
- Signature: `bool ReadDirectory(vector<dir_entry> &vec)` (SDL only)
- Purpose: List directory contents with metadata
- Inputs: vec (output)
- Outputs/Return: true if read, vec sorted (directories first, then alphabetically by name)
- Side effects: None
- Calls: None visible
- Notes: dir_entry holds name, size, is_directory, is_volume flags

## Control Flow Notes
Utility layer providing file operations on-demand (not loop-bound). Typical usage: create FileSpecifier ΓåÆ check Exists() / GetType() ΓåÆ Open() ΓåÆ read/write ΓåÆ auto-close. Resource fork access adds Push()/Pop() on Mac to manage global resource context. Supports game asset loading (maps, physics, shapes, sounds) and save/load cycles.

## External Dependencies
- **<vector>, <string>** (STL): Dynamic collections for paths and directory listings
- **<SDL.h>** (SDL): `SDL_RWops` for cross-platform file I/O abstraction
- **Mac/Carbon APIs**: `FSSpec`, `OSErr`, `Handle`, `OSType` (Mac/Carbon.h)
- **tags.h**: `Typecode` enum, `FOUR_CHARS_TO_INT` macro, typecode access functions
- **<time.h>**: `time_t` / `TimeType` for file modification dates
- **Platform conditionals**: `#ifdef mac`, `#ifdef SDL`, `#ifdef __WIN32__` select implementations
