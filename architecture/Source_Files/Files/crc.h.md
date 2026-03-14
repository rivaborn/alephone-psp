# Source_Files/Files/crc.h

## File Purpose
Header file declaring CRC (Cyclic Redundancy Check) calculation functions for the engine. Provides utilities to compute 32-bit and 16-bit checksums of files and raw data buffers for integrity validation.

## Core Responsibilities
- Declare 32-bit CRC calculation for files (by FileSpecifier and OpenedFile)
- Declare 32-bit CRC calculation for raw data buffers
- Declare 16-bit CCITT CRC calculation for data buffers
- Define interface between file I/O layer and CRC computation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| FileSpecifier | class (forward decl.) | Represents a file path/location; used to identify files for CRC computation |
| OpenedFile | class (forward decl.) | Represents an open file handle; used for CRC of already-opened files |

## Global / File-Static State
None

## Key Functions / Methods

### calculate_crc_for_file
- Signature: `uint32 calculate_crc_for_file(FileSpecifier& File)`
- Purpose: Compute 32-bit CRC of an entire file identified by FileSpecifier
- Inputs: FileSpecifier reference (file to checksum)
- Outputs/Return: 32-bit unsigned CRC value
- Side effects: Likely opens, reads, and closes the file
- Calls: Not inferable from this file
- Notes: Implementation elsewhere; used for file integrity checks

### calculate_crc_for_opened_file
- Signature: `uint32 calculate_crc_for_opened_file(OpenedFile& OFile)`
- Purpose: Compute 32-bit CRC of an already-opened file
- Inputs: OpenedFile reference (open file handle)
- Outputs/Return: 32-bit unsigned CRC value
- Side effects: Likely reads from current file position; may reset position
- Calls: Not inferable from this file
- Notes: Alternative to `calculate_crc_for_file`; avoids redundant file open/close

### calculate_data_crc
- Signature: `uint32 calculate_data_crc(unsigned char *buffer, long length)`
- Purpose: Compute 32-bit CRC of raw memory buffer
- Inputs: Buffer pointer, buffer length in bytes
- Outputs/Return: 32-bit unsigned CRC value
- Side effects: None (read-only of buffer)
- Calls: Not inferable from this file
- Notes: Generic data CRC; used for non-file data integrity checks

### calculate_data_crc_ccitt
- Signature: `uint16 calculate_data_crc_ccitt(unsigned char *buffer, long length)`
- Purpose: Compute 16-bit CCITT CRC of raw memory buffer
- Inputs: Buffer pointer, buffer length in bytes
- Outputs/Return: 16-bit unsigned CCITT CRC value
- Side effects: None (read-only of buffer)
- Calls: Not inferable from this file
- Notes: Different CRC variant (CCITT); produces 16-bit checksum

## Control Flow Notes
Utility module. Functions are called on-demand from game logic and I/O layers (not part of main update/render loop). Likely used during file loading, data validation, and save-file integrity checks.

## External Dependencies
- **Forward declarations:** `FileSpecifier`, `OpenedFile` (defined elsewhere, likely in file handling layer)
- **Primitive types:** `uint32`, `uint16`, `long` (defined in platform headers)
- **History note (Aug 15, 2000):** Refactored to use object-oriented file handler (FileSpecifier, OpenedFile)
