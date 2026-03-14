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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `wad_header` | struct | File header: version, checksum, directory offset, entry counts, sizes for different WAD format versions |
| `directory_entry` / `old_directory_entry` | struct | Index entry mapping: file offset and length of each WAD block; newer version includes overlay index |
| `entry_header` / `old_entry_header` | struct | Entry header in WAD data stream: tag, next-offset (linked-list), length, optional offset field |
| `tag_data` | struct | In-memory metadata for one data block: tag type, pointer to data, length, offset (for patches) |
| `wad_data` | struct | In-memory WAD: tag count, read-only flag, array of tag_data entries |
| `OpenedFile` | class | File handle abstraction (from FileHandler.h); read/write/seek operations |
| `FileSpecifier` | class | File path abstraction (from FileHandler.h); filesystem operations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `BetweenLevels` | bool | global | Flag: if true, use `level_transition_malloc()` for WAD allocation (level-specific memory); if false, use standard `malloc()` (e.g., for 3D model loading) |
| `internal_data[MAXIMUM_OPEN_WADFILES]` | wad_internal_data* [3] | file-static | Array of open WAD file state; holds up to 3 simultaneously open files (unused in current code) |

## Key Functions / Methods

### read_wad_header
- **Signature**: `bool read_wad_header(OpenedFile& OFile, struct wad_header *header)`
- **Purpose**: Read and validate WAD file header from disk; extract metadata for subsequent operations
- **Inputs**: `OFile` (opened file handle), `header` (pointer to receive header data)
- **Outputs/Return**: `true` if header read and version valid; `false` on read error or unsupported version
- **Side effects**: Reads 128 bytes from file offset 0; sets game error on failure
- **Calls**: `read_from_file()`, `unpack_wad_header()`, `set_game_error()`
- **Notes**: Validates `header->version <= CURRENT_WADFILE_VERSION`, `data_version <= 2`, `wad_count >= 1`; aborts if any check fails

### read_indexed_wad_from_file
- **Signature**: `struct wad_data *read_indexed_wad_from_file(OpenedFile& OFile, struct wad_header *header, short index, bool read_only)`
- **Purpose**: Main API to load a single WAD (map, texture set, etc.) from a file by index; return in-memory representation
- **Inputs**: `OFile` (opened file), `header` (file header), `index` (0-based WAD index), `read_only` (if true, map data in-place; else copy to new allocations)
- **Outputs/Return**: Pointer to `wad_data` structure, or NULL on allocation failure
- **Side effects**: Allocates memory via `level_transition_malloc()` or `malloc()`; padding added for Marathon 1 compatibility; sets game error code on failure
- **Calls**: `size_of_indexed_wad()`, `read_indexed_wad_from_file_into_buffer()`, `convert_wad_from_raw()` or `convert_wad_from_raw_modifiable()`, `free()`, `set_game_error()`
- **Notes**: Allocates 2├ù WAD size if modifiable (raw copy + structured copy); read-only WADs point directly into file buffer for efficiency

### extract_type_from_wad
- **Signature**: `void *extract_type_from_wad(struct wad_data *wad, WadDataType type, size_t *length)`
- **Purpose**: Query API: find and return data block by tag type from a loaded WAD
- **Inputs**: `wad` (loaded WAD), `type` (tag to search for, e.g., `MAP_INFO_TAG`), `length` (pointer to receive size)
- **Outputs/Return**: Pointer to data if found; NULL if tag not present; sets `*length` to data size (or 0 if not found)
- **Side effects**: None
- **Calls**: Linear search through `wad->tag_data[index]`
- **Notes**: Returns pointer to data without copying; caller must not modify if `read_only_data` is set

### append_data_to_wad
- **Signature**: `struct wad_data *append_data_to_wad(struct wad_data *wad, WadDataType type, void *data, size_t size, size_t offset)`
- **Purpose**: Add or replace a data block in a WAD; reallocate tag array if necessary
- **Inputs**: `wad` (WAD to modify, must not be read-only), `type` (tag), `data` (bytes to add), `size` (byte count), `offset` (optional offset for patch application)
- **Outputs/Return**: Pointer to modified `wad` (same as input)
- **Side effects**: Calls `malloc()` / `free()` to grow/shrink tag array and copy data; alerts user on allocation failure
- **Calls**: `malloc()`, `free()`, `memcpy()`, `alert_user()`, `objlist_clear()`, `objlist_copy()`
- **Notes**: If tag already exists, old data freed and replaced; new data always copied; asserts that size > 0 and WAD is not read-only

### write_wad
- **Signature**: `bool write_wad(OpenedFile& OFile, struct wad_header *file_header, struct wad_data *wad, long offset)`
- **Purpose**: Write WAD data (header chain) to file at given offset
- **Inputs**: `OFile` (opened file for writing), `file_header` (header defining entry size), `wad` (in-memory data to write), `offset` (starting file position)
- **Outputs/Return**: `true` if all writes succeeded; `false` on file error
- **Side effects**: Writes to file; updates entry offsets and next_offset pointers
- **Calls**: `write_to_file()`, `pack_old_entry_header()` or `pack_entry_header()`, `get_entry_header_length()`
- **Notes**: Builds linked-list chain of entry headers; last entry has `next_offset = 0`

### create_empty_wad
- **Signature**: `struct wad_data *create_empty_wad(void)`
- **Purpose**: Allocate and initialize an empty WAD in memory for building new data
- **Inputs**: None
- **Outputs/Return**: Pointer to newly allocated (zeroed) `wad_data`, or NULL on allocation failure
- **Side effects**: Calls `malloc()`
- **Calls**: `malloc()`, `obj_clear()`
- **Notes**: Returned WAD is modifiable; caller should populate with `append_data_to_wad()`

### calculate_and_store_wadfile_checksum
- **Signature**: `void calculate_and_store_wadfile_checksum(OpenedFile& OFile)`
- **Purpose**: Compute and store CRC32 checksum of entire WAD file for integrity verification
- **Inputs**: `OFile` (opened file for read/write)
- **Outputs/Return**: None
- **Side effects**: Reads entire file, writes checksum to header field at offset 0
- **Calls**: `read_wad_header()`, `write_wad_header()`, `calculate_crc_for_opened_file()`
- **Notes**: Temporarily zeros checksum field before calculation (checksum does not include itself)

### get_flat_data / inflate_flat_data
- **Signature**: `void *get_flat_data(FileSpecifier& File, bool use_union, short wad_index)` / `struct wad_data *inflate_flat_data(void *data, struct wad_header *header)`
- **Purpose**: Serialize a WAD for network transmission; deserialize received flat data back into `wad_data`
- **Inputs** (get): `File` (WAD file), `use_union` (unused, must be false), `wad_index` (which WAD to flatten)
- **Inputs** (inflate): `data` (flat binary blob), `header` (to receive parsed header)
- **Outputs/Return**: `get_flat_data` returns malloc'd buffer; `inflate_flat_data` returns parsed `wad_data` or NULL
- **Side effects**: Allocates memory; unpacks encapsulated header with magic cookie and length
- **Calls**: `get_flat_data`: `open_wad_file_for_reading()`, `read_wad_header()`, `size_of_indexed_wad()`, `read_indexed_wad_from_file_into_buffer()`, `close_wad_file()` | `inflate_flat_data`: `unpack_wad_header()`, `convert_wad_from_raw()`
- **Notes**: Encapsulates magic cookie (CURRENT_FLAT_MAGIC_COOKIE), total length, and header before raw WAD data; used for net transfer or caching

### Packing/Unpacking Functions
(Summary; detailed per-type functions exist for `wad_header`, `directory_entry`, `old_directory_entry`, `entry_header`, `old_entry_header`)
- **Signature**: `static uint8 *unpack_*(...) / static uint8 *pack_*(...)`
- **Purpose**: Convert binary byte stream Γåö native C structs; handle endianness (big-endian file format)
- **Inputs**: Byte stream pointer, object pointer(s), count
- **Outputs/Return**: Advanced stream pointer
- **Side effects**: Calls `StreamToValue()` / `ValueToStream()` macros for individual fields
- **Notes**: Assert correct total bytes consumed; work for both single and array packing

## Control Flow Notes
**Initialization (typically at game startup or level load):**
- Engine calls `read_indexed_wad_from_file()` to load a map or other asset WAD
- For read-only access (most cases), WAD is parsed in-place; for modification, data is copied

**Modification (level authoring or patching):**
- Create empty WAD with `create_empty_wad()`
- Add data blocks via `append_data_to_wad()` / `remove_tag_from_wad()`
- Write to disk via `write_wad_header()`, `write_directorys()`, `write_wad()`
- Finalize with `calculate_and_store_wadfile_checksum()`

**Network transfer (multiplayer):**
- Flatten WAD with `get_flat_data()` before transmission
- Recipient unpacks via `inflate_flat_data()` and uses as normal `wad_data`

**Teardown:**
- `free_wad()` (not shown in this file, assumed in wad.h) deallocates `wad_data` and owned pointers

**Control flow integration:**
This is a **data-loading and serialization layer**, not a frame/update/render loop component. It is called on demand (asset loading, save/load, network sync) rather than per-frame.

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
