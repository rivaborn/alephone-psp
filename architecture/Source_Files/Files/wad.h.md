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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | 128-byte file header; stores version, checksum, directory offset, WAD count, file metadata |
| `directory_entry` | struct | 10-byte directory entry; maps WAD index to file offset, length, and in-place modification index |
| `old_directory_entry` | struct | 8-byte legacy directory entry (version compatibility) |
| `entry_header` | struct | 16-byte per-tag header; stores tag ID, length, next offset, expansion offset |
| `old_entry_header` | struct | 12-byte legacy entry header (version compatibility) |
| `tag_data` | struct | In-memory tag representation; tag ID, data pointer, length, patch offset |
| `wad_data` | struct | Complete in-memory WAD; tag count and array of `tag_data` elements |
| `WadDataType` | typedef | `uint32`; 4-byte tag identifier (e.g., `POINT_TAG`, `POLYGON_TAG`) |

## Global / File-Static State
None defined in this header. State is encapsulated in `OpenedFile` and `FileSpecifier` objects passed by reference, and a static "between levels" flag via `SetBetweenlevels()` / `IsBetweenLevels()`.

## Key Functions / Methods

### open_wad_file_for_reading / open_wad_file_for_writing
- Signature: `bool open_wad_file_for_reading/writing(FileSpecifier& File, OpenedFile& OFile)`
- Purpose: Open a WAD file for reading or writing; associate it with an `OpenedFile` handle
- Inputs: `FileSpecifier` (file to open), output `OpenedFile` reference
- Outputs/Return: `bool` success/failure
- Side effects: Opens file descriptor, may allocate memory in `OpenedFile`
- Calls: Not inferable from this file

### read_indexed_wad_from_file
- Signature: `struct wad_data *read_indexed_wad_from_file(OpenedFile& OFile, struct wad_header *header, short index, bool read_only)`
- Purpose: Load a specific WAD (by index) from an open file into memory
- Inputs: Open file handle, header (contains WAD metadata), index (0ΓÇô`wad_count`), read-only flag
- Outputs/Return: Pointer to in-memory `wad_data` struct (heap-allocated)
- Side effects: Allocates memory for `wad_data` and `tag_data` array; I/O from file
- Calls: Calls `free_wad()` implicitly to clean up on error (not directly visible)
- Notes: Returned data must be freed with `free_wad()`. Read-only flag sets `read_only_data` pointer for mmap-like behavior

### extract_type_from_wad
- Signature: `void *extract_type_from_wad(struct wad_data *wad, WadDataType type, size_t *length)`
- Purpose: Find and extract a tagged data element from a loaded WAD by tag ID
- Inputs: Loaded WAD, tag to search for, output length pointer
- Outputs/Return: Pointer to tag data (or NULL if not found), length written to output parameter
- Side effects: None (read-only)
- Calls: Not inferable
- Notes: Returns pointer into existing `wad_data`, not a copy; caller must not free result

### append_data_to_wad
- Signature: `struct wad_data *append_data_to_wad(struct wad_data *wad, WadDataType type, void *data, size_t size, size_t offset)`
- Purpose: Add or replace a tagged data entry in a WAD; supports in-place expansion via offset
- Inputs: WAD to modify, tag ID, data buffer, size, expansion offset (for in-place growth)
- Outputs/Return: Updated/new `wad_data` struct pointer
- Side effects: Allocates new `tag_data` entry; may reallocate `wad_data` array
- Calls: Not inferable
- Notes: Offset parameter supports "in-place expansion" of existing data (for level editors)

### write_wad
- Signature: `bool write_wad(OpenedFile& OFile, struct wad_header *file_header, struct wad_data *wad, long offset)`
- Purpose: Write a single WAD to an open file at a specified offset
- Inputs: Open file, header (for WAD metadata), WAD data, file offset
- Outputs/Return: `bool` success/failure
- Side effects: File I/O; writes all tag entries
- Calls: Not inferable
- Notes: Offset allows fragmented writing or overwriting

### get_flat_data / inflate_flat_data
- Signature: `void *get_flat_data(FileSpecifier& File, bool use_union, short wad_index)` ΓåÆ `struct wad_data *inflate_flat_data(void *data, struct wad_header *header)`
- Purpose: Serialize/deserialize a single WAD for transfer or storage (encapsulation)
- Inputs: File, optional union-mode flag, WAD index; then flat buffer and header
- Outputs/Return: Opaque flat buffer (length via `get_flat_data_length()`); then reconstructed `wad_data`
- Side effects: Allocation for flat buffer; inflation allocates `wad_data`
- Calls: Not inferable

### SetBetweenlevels / IsBetweenLevels
- Signature: `void SetBetweenlevels(bool _BetweenLevels)` / `bool IsBetweenLevels()`
- Purpose: Control WAD memory allocator behavior across level transitions
- Inputs: Boolean state
- Outputs/Return: Current state (getter)
- Side effects: Affects global allocator behavior (prevents interference with loaded models)
- Notes: Default is true (between levels); set to false when loading model files with WAD format outside level load

## Control Flow Notes
WAD system lifecycle:
1. **Initialization**: `create_wadfile()` or `open_wad_file_for_reading/writing()` ΓåÆ `read_wad_header()` to validate
2. **Data Loading**: `read_indexed_wad_from_file()` loads entire WAD into memory; game extracts specific tags via `extract_type_from_wad()`
3. **In-Memory Manipulation**: `append_data_to_wad()` / `remove_tag_from_wad()` for editing; `calculate_wad_length()` for size queries
4. **Serialization**: `get_flat_data()` / `inflate_flat_data()` for transfer; `write_wad()` / `write_directorys()` for persistence
5. **Cleanup**: `free_wad()` at level end or shutdown
6. **Level Transitions**: `SetBetweenlevels(true)` to protect level-specific allocations from model loading

## External Dependencies
- **Included**: `tags.h` ΓÇö defines `Typecode` enum and tag constants (`POINT_TAG`, `POLYGON_TAG`, `MAP_INFO_TAG`, etc.)
- **Defined elsewhere**: 
  - `FileSpecifier` class ΓÇö file path abstraction layer
  - `OpenedFile` class ΓÇö file handle/stream abstraction
  - Standard C types (`uint32`, `int32`, `int16`, `byte`, `long`)
  - `FOUR_CHARS_TO_INT` macro (likely from `tags.h` or `cstypes.h`)
