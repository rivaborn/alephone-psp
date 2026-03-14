# Source_Files/Files/crc.cpp

## File Purpose
Implements CRC-32 and CRC-CCITT checksum generation for files and memory buffers in the Aleph One game engine. Provides table-based (lookup) CRC computation optimized for file integrity validation and data checksumming.

## Core Responsibilities
- Calculate CRC-32 checksums for unopened files (FileSpecifier) and opened file handles (OpenedFile)
- Calculate CRC-32 checksums for raw memory buffers
- Calculate CRC-CCITT checksums (non-reversed CCITT polynomial 0x1021)
- Manage dynamic allocation/deallocation of CRC-32 lookup table (lazy initialization)
- Handle chunked file I/O to support large files with fixed buffer size

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| crc_table | uint32* (static) | Dynamically allocated 256-entry lookup table for CRC-32 polynomial computations |
| crc_ccitt_table | uint16[256] (static const) | Precomputed 256-entry lookup table for CRC-CCITT calculations |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| crc_table | uint32* | static | CRC-32 lookup table; allocated on-demand, freed after each CRC calculation |
| crc_ccitt_table | uint16[256] | static const | Precomputed CCITT polynomial lookup table (no allocation) |
| BUFFER_SIZE | #define 1024 | compile-time | Chunk size for file read operations |
| CRC32_POLYNOMIAL | #define 0xEDB88320L | compile-time | CRC-32 polynomial for table generation |
| TABLE_SIZE | #define 256 | compile-time | Lookup table entry count |

## Key Functions / Methods

### calculate_crc_for_file
- Signature: `uint32 calculate_crc_for_file(FileSpecifier& File)`
- Purpose: Public entry point; calculate CRC-32 for a file by path specification
- Inputs: FileSpecifier reference (unopened file)
- Outputs/Return: uint32 CRC value; 0 if file open fails
- Side effects: Opens and closes file; allocates/deallocates CRC table via `build_crc_table()` / `free_crc_table()`
- Calls: File.Open(), calculate_crc_for_opened_file(), OFile.Close()
- Notes: Delegates to calculate_crc_for_opened_file() after opening

### calculate_crc_for_opened_file
- Signature: `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- Purpose: Public entry point; calculate CRC-32 for an already-opened file
- Inputs: OpenedFile reference
- Outputs/Return: uint32 CRC; 0 on buffer allocation failure
- Side effects: Heap-allocates temporary buffer (BUFFER_SIZE); allocates/frees crc_table
- Calls: build_crc_table(), calculate_file_crc(), free_crc_table()
- Notes: Asserts on buffer allocation failure

### calculate_data_crc
- Signature: `uint32 calculate_data_crc(unsigned char *buffer, long length)`
- Purpose: Public entry point; calculate CRC-32 for raw memory buffer
- Inputs: Byte pointer, length in bytes
- Outputs/Return: uint32 CRC; 0 on table allocation failure
- Side effects: Allocates/frees crc_table; asserts on null buffer input
- Calls: build_crc_table(), calculate_buffer_crc(), free_crc_table()
- Notes: Uses XOR initialization/finalization (0xFFFFFFFFL) for consistency with file-based CRC

### calculate_file_crc
- Signature: `static uint32 calculate_file_crc(unsigned char *buffer, short buffer_size, OpenedFile& OFile)`
- Purpose: Core implementation; compute CRC-32 by chunked file reading
- Inputs: Read buffer (preallocated), buffer size, opened file reference
- Outputs/Return: uint32 CRC; 0 on any I/O or position error
- Side effects: Reads entire file sequentially; saves and restores original file position
- Calls: OFile.GetPosition(), OFile.GetLength(), OFile.SetPosition(), OFile.Read(), calculate_buffer_crc()
- Notes: Preserves file seek position for integration with other I/O operations; fail-safe on all I/O errors

### calculate_buffer_crc
- Signature: `static uint32 calculate_buffer_crc(long count, uint32 crc, void *buffer)`
- Purpose: Core algorithm; incremental byte-by-byte CRC-32 update
- Inputs: Byte count, current CRC state, void pointer to data
- Outputs/Return: Updated uint32 CRC
- Side effects: None (read-only access to crc_table and buffer)
- Calls: None (table lookup only)
- Notes: Assumes crc_table is allocated; processes one byte per iteration

### build_crc_table
- Signature: `static bool build_crc_table(void)`
- Purpose: Lazily allocate and compute the 256-entry CRC-32 lookup table
- Inputs: None
- Outputs/Return: bool (true if allocation and computation succeeded)
- Side effects: Allocates crc_table global; asserts if already allocated
- Calls: None
- Notes: Standard polynomial bit-rotation algorithm (8 iterations per entry); called in pairs with free_crc_table()

### free_crc_table
- Signature: `static void free_crc_table(void)`
- Purpose: Deallocate the CRC-32 lookup table
- Inputs: None
- Outputs/Return: None
- Side effects: Deletes crc_table and nulls it; asserts if not allocated
- Calls: None
- Notes: Always paired with build_crc_table()

### calculate_data_crc_ccitt
- Signature: `uint16 calculate_data_crc_ccitt(unsigned char *data, long length)`
- Purpose: Public entry point; calculate CRC-CCITT for memory buffer
- Inputs: Byte pointer, length in bytes
- Outputs/Return: uint16 CRC-CCITT result
- Side effects: None (statically precomputed table)
- Calls: None
- Notes: Adapted from Jack Klein's public domain code; uses precomputed table (no allocation overhead)

## Control Flow Notes
**CRC-32 calculation sequence:** Public entry ΓåÆ build_crc_table() ΓåÆ (file I/O or direct buffer processing) ΓåÆ calculate_buffer_crc() loop ΓåÆ free_crc_table() ΓåÆ return

**CRC-CCITT calculation:** Direct table lookup loop; no allocation or I/O overhead

**File-position preservation:** calculate_file_crc() saves initial position before I/O, restores afterΓÇödesigned to integrate with larger file workflows without disrupting seek state

## External Dependencies
- **FileHandler.h**: OpenedFile, FileSpecifier classes (file I/O abstraction)
- **cseries.h**: Type definitions (uint32, uint16, byte), platform macros
- Standard library: new/delete (stdlib.h), assert macro
- **Defined elsewhere:** OpenedFile::Open(), Close(), GetPosition(), SetPosition(), GetLength(), Read(); FileSpecifier::Open()
