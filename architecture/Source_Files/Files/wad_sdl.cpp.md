# Source_Files/Files/wad_sdl.cpp

## File Purpose
Provides SDL-based file discovery functions for locating map files (WAD files) by checksum or modification date within a configurable search path. Implements searchable catalog functionality for the Aleph One game engine's resource loading system.

## Core Responsibilities
- Search for map files across multiple directories by checksum validation
- Locate files by modification date across the search path
- Abstract checksum and date matching logic via FileFinder subclasses
- Iterate through `data_search_path` directories to find resource files
- Read and parse WAD file checksums at fixed offset (0x44)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FindByChecksum` | class | FileFinder subclass that searches for files matching a specific 32-bit checksum |
| `FindByDate` | class | FileFinder subclass that searches for files matching a specific modification date |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `data_search_path` | `vector<DirectorySpecifier>` | extern global | Search directories for locating resource files (defined in shell_sdl.cpp) |

## Key Functions / Methods

### find_wad_file_that_has_checksum
- Signature: `bool find_wad_file_that_has_checksum(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, uint32 checksum)`
- Purpose: Locate a WAD file with a specific checksum value across the data search path
- Inputs: 
  - `matching_file`: output reference to store found file
  - `file_type`: file typecode filter (unused in implementation)
  - `path_resource_id`: path resource ID (unused in implementation)
  - `checksum`: target 32-bit checksum to match
- Outputs/Return: `true` if file found, `false` otherwise; matching file written to `matching_file` reference
- Side effects: Opens files to read checksum data; modifies `matching_file` if successful
- Calls: `FindByChecksum` constructor, `FindByChecksum::Find()`, iterator operations on `data_search_path`
- Notes: Iterates through all directories; stops at first match; checksum read from offset 0x44 using big-endian format

### find_file_with_modification_date
- Signature: `bool find_file_with_modification_date(FileSpecifier &matching_file, Typecode file_type, short path_resource_id, TimeType modification_date)`
- Purpose: Locate a file with a specific modification date across the data search path
- Inputs:
  - `matching_file`: output reference to store found file
  - `file_type`: file typecode filter (unused in implementation)
  - `path_resource_id`: path resource ID (unused in implementation)
  - `modification_date`: target modification timestamp to match
- Outputs/Return: `true` if file found, `false` otherwise; matching file written to `matching_file`
- Side effects: Queries file metadata; modifies `matching_file` if successful
- Calls: `FindByDate` constructor, `FindByDate::Find()`, iterator operations on `data_search_path`
- Notes: Iterates through all directories; stops at first match

## Control Flow Notes
These functions are integration points for resource discovery at startup or on-demand file loading. The pattern is: instantiate a finder, iterate directories from `data_search_path`, call `Find()` for each directory, and return on first match. Both functions follow identical iteration patterns but with different matcher criteria.

## External Dependencies
- `#include "cseries.h"` ΓÇö common type definitions and platform macros
- `#include "FileHandler.h"` ΓÇö `FileSpecifier`, `OpenedFile`, `DirectorySpecifier`, `TimeType` classes
- `#include "find_files.h"` ΓÇö `FileFinder` base class, `Typecode` type
- `#include <SDL_endian.h>` ΓÇö `SDL_ReadBE32()` for big-endian checksum reads
- **Defined elsewhere**: `data_search_path` (extern vector from shell_sdl.cpp)
