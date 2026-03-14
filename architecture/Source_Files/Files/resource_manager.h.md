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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (fwd) | Represents a file path/location for resource file opening |
| LoadedResource | class (fwd) | Container for loaded resource data |
| SDL_RWops | extern struct | SDL abstraction over file I/O streams |

## Global / File-Static State
None.

## Key Functions / Methods

### initialize_resources
- Signature: `void initialize_resources(void)`
- Purpose: One-time setup for resource system
- Inputs: None
- Outputs/Return: None
- Side effects: Initializes internal resource tables/caches
- Calls: Not inferable
- Notes: Must be called before other resource operations

### open_res_file / close_res_file
- Signature: `SDL_RWops *open_res_file(FileSpecifier &file)`, `void close_res_file(SDL_RWops *file)`
- Purpose: Lifecycle management for resource file handles
- Inputs: FileSpecifier (path), or SDL_RWops (handle to close)
- Outputs/Return: RWops handle to opened file
- Side effects: Allocates/deallocates I/O stream
- Calls: Not inferable
- Notes: Caller must close to avoid leaks

### use_res_file / cur_res_file
- Signature: `void use_res_file(SDL_RWops *file)`, `SDL_RWops *cur_res_file(void)`
- Purpose: Set and retrieve the currently active resource file for lookups
- Inputs: RWops handle
- Outputs/Return: Current active file handle
- Side effects: Changes resource lookup context
- Calls: Not inferable
- Notes: Subsequent `get_resource()` calls search the active file

### count_resources / get_resource_id_list
- Signature: `size_t count_resources(uint32 type)`, `void get_resource_id_list(uint32 type, vector<int> &ids)`
- Purpose: Enumerate resources of a given type
- Inputs: 4-byte resource type code
- Outputs/Return: Count or populated ID vector
- Side effects: Fills output vector
- Calls: Not inferable
- Notes: "count_1_" variants limit to single file; default variants search hierarchy

### get_resource / has_resource
- Signature: `bool get_resource(uint32 type, int id, LoadedResource &rsrc)`, `bool has_resource(uint32 type, int id)`
- Purpose: Fetch/check resource by type and ID
- Inputs: Type code, resource ID
- Outputs/Return: Success flag; rsrc filled on success
- Side effects: Allocates resource data into LoadedResource
- Calls: Not inferable
- Notes: Supports direct ID lookup; "1" variants search single file only

### get_ind_resource
- Signature: `bool get_ind_resource(uint32 type, int index, LoadedResource &rsrc)`
- Purpose: Fetch resource by type and ordinal index
- Inputs: Type code, 0-based index
- Outputs/Return: Success flag; rsrc filled on success
- Side effects: Allocates resource data into LoadedResource
- Calls: Not inferable
- Notes: Useful when iterating over all resources of a type

## Control Flow Notes
Likely invoked during engine initialization (`initialize_resources`) and asset load phases (level/map loading). Maintains a current resource file context for lookups, enabling hierarchical searches (local overrides ΓåÆ defaults). The "1" variants suggest support for resource fork merging or override chains.

## External Dependencies
- `stdio.h` ΓÇô Basic I/O (likely for legacy reasons; SDL provides RWops)
- `<vector>` ΓÇô Dynamic ID/resource lists
- `<SDL.h>` ΓÇô RWops file abstraction
- **Forward declared:** FileSpecifier, LoadedResource (defined in separate headers)
- **uint32** ΓÇô Assumed from stdint.h or engine-specific typedef
