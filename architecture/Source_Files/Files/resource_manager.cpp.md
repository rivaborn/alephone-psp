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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `res_file_t` | struct | Open resource file with parsed resource map; wraps SDL_RWops and nested maps |
| `id_map_t` | typedef (map) | Maps resource ID ΓåÆ byte offset to resource data |
| `type_map_t` | typedef (map) | Maps resource type (4-char code) ΓåÆ id_map_t |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `res_file_list` | `static list<res_file_t *>` | static | List of all open resource files |
| `cur_res_file_t` | `static list<res_file_t *>::iterator` | static | Current resource file (top of stack) |

## Key Functions / Methods

### open_res_file
- Signature: `SDL_RWops *open_res_file(FileSpecifier &file)`
- Purpose: Main entry point; opens a resource file, trying multiple naming conventions
- Inputs: FileSpecifier reference (base file path)
- Outputs/Return: SDL_RWops pointer to opened file, or NULL on failure
- Side effects: Parses resource map; adds `res_file_t` to `res_file_list`; updates `cur_res_file_t`
- Calls: `open_res_file_from_path()`, `open_res_file_from_rwops()`, platform-specific fork detection (BeOS, macOS)
- Notes: Tries suffixes `.rsrc`, `.resources`, raw file, `/..namedfork/rsrc`, then platform forks

### open_res_file_from_rwops
- Signature: `SDL_RWops *open_res_file_from_rwops(SDL_RWops *f)`
- Purpose: Wrap an already-opened SDL_RWops as a managed resource file
- Inputs: SDL_RWops pointer (must be opened)
- Outputs/Return: Same SDL_RWops pointer on success, NULL on parse failure
- Side effects: Calls `read_map()`; allocates and adds `res_file_t` to stack
- Calls: `res_file_t::read_map()`, `SDL_RWclose()`, logging functions

### res_file_t::read_map
- Signature: `bool res_file_t::read_map(void)`
- Purpose: Parse resource map from file, handling format detection transparently
- Inputs: None (reads from member `f`)
- Outputs/Return: bool (success)
- Side effects: Populates nested `types` map with type/ID/offset entries; seeks file
- Calls: `is_applesingle()`, `is_macbinary()`, `SDL_RWseek()`, `SDL_ReadBE32()`, `SDL_ReadBE16()`
- Notes: Validates header integrity and fork offsets; logs format type detected; handles 128-byte MacBinary header alignment

### get_resource
- Signature: `bool get_resource(uint32 type, int id, LoadedResource &rsrc)`
- Purpose: Get resource by type/ID, iterating from current file backward through stack
- Inputs: 4-char resource type, resource ID, LoadedResource reference to populate
- Outputs/Return: bool (found)
- Side effects: Allocates memory via `malloc()`; updates LoadedResource with pointer and size
- Calls: `(*cur_res_file_t)->get_resource()` and previous files in reverse
- Notes: Stops on first match; caller responsible for freeing via LoadedResource dtor

### res_file_t::get_resource
- Signature: `bool res_file_t::get_resource(uint32 type, int id, LoadedResource &rsrc) const`
- Purpose: Retrieve resource from this specific file
- Inputs: type, ID, LoadedResource ref
- Outputs/Return: bool (found)
- Side effects: Allocates memory, reads file, populates LoadedResource
- Calls: `SDL_RWseek()`, `SDL_ReadBE32()`, `SDL_RWread()`, `malloc()`, `rsrc.SetData()` or direct member assignment
- Notes: Reads 4-byte BE size prefix; platform-dependent assignment (mac uses `SetData()`, SDL uses raw pointer/size)

### count_resources / count_1_resources
- Signature: `size_t count_resources(uint32 type)` / `size_t count_1_resources(uint32 type)`
- Purpose: Count resources of type across all (or current) open files
- Inputs: 4-char type code
- Outputs/Return: count (size_t)
- Calls: `(*cur_res_file_t)->count_resources()`

### get_resource_id_list / get_1_resource_id_list
- Signature: `void get_resource_id_list(uint32 type, vector<int> &ids)` (and `_1_` variant)
- Purpose: Populate vector with IDs of all resources of given type
- Inputs: type code, output vector reference
- Side effects: Clears input vector; appends IDs
- Calls: `(*cur_res_file_t)->get_resource_id_list()`

### has_resource / has_1_resource
- Signature: `bool has_resource(uint32 type, int id)` (and `_1_` variant)
- Purpose: Check presence of resource without loading
- Inputs: type code, ID
- Outputs/Return: bool (exists)
- Calls: Stack iteration or current-file lookup

### Utility functions (private)
- `is_applesingle()` ΓÇô Detects AppleSingle format header; extracts fork offset/length
- `is_macbinary()` ΓÇô Detects MacBinary II/III header; validates CRC; extracts fork sizes
- `find_res_file_t()` ΓÇô Locates res_file_t by SDL_RWops pointer in list

## Control Flow Notes
**Startup:** `initialize_resources()` (no-op); files opened on demand via `open_res_file()`.  
**Resource queries:** Default to current file (`*_1_*`) or iterate backward through stack (non-`_1_`).  
**Lifecycle:** Open ΓåÆ parse map ΓåÆ populate type/ID tables ΓåÆ query/load resources ΓåÆ close.  
**Format handling:** Format detection is transparent; `.read_map()` handles all three formats with unified on-disk offset calculations.

## External Dependencies
- **SDL_RWops** (SDL_rwops.h) ΓÇô file abstraction; endian read functions (SDL_ReadBE32, etc.)
- **FileHandler.h** ΓÇô FileSpecifier, LoadedResource, platform-specific file operations
- **Logging.h** ΓÇô logNote, logTrace, logAnomaly, logDump4 macros
- **cseries.h** ΓÇô uint32, uint16, uint8 typedef; platform macros (__MACOS__, __BEOS__)
- **csfiles_beos.cpp** (defined elsewhere) ΓÇô `has_rfork_attribute()`, `sdl_rw_from_rfork()`
- **mac_rwops.h** (macOS only) ΓÇô `open_fork_from_existing_path()`
