# Source_Files/Files/find_files_sdl.cpp

## File Purpose
SDL implementation of recursive file searching for the Aleph One game engine. Provides a template-method pattern where `FileFinder::Find()` handles directory traversal and type filtering, while derived classes override `found()` to process matching files.

## Core Responsibilities
- Recursively traverse directories to locate files matching a given type code
- Sort directory entries (directories before files, then alphabetically)
- Filter files by type code or accept wildcard matches
- Support early termination on first match or continue-all-files collection via virtual callback
- Construct full file paths during traversal using `FileSpecifier` operators

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileFinder` | class | Base class defining the recursive search template method |
| `FindAllFiles` | class | Derived class that collects all matching files into a vector |
| `dir_entry` | struct | Directory entry (from FileHandler.h); supports sorting with directories before files |
| `FileSpecifier` | class | File path abstraction; supports path concatenation via `operator+` |
| `DirectorySpecifier` | class | Directory path abstraction |

## Global / File-Static State
None.

## Key Functions / Methods

### FileFinder::Find
- **Signature:** `bool Find(DirectorySpecifier &dir, Typecode type, bool recursive)`
- **Purpose:** Template-method recursively searching a directory tree for files of a given type.
- **Inputs:** 
  - `dir` ΓÇö directory to search
  - `type` ΓÇö file type code to match (`WILDCARD_TYPE` = any type)
  - `recursive` ΓÇö whether to descend into subdirectories
- **Outputs/Return:** `true` if `found()` returned true for any file (search terminated early); `false` if no matches or search exhausted.
- **Side effects:** Calls virtual `found()` for each matching file; filesystem reads via `ReadDirectory()`.
- **Calls:** `dir.ReadDirectory()`, `std::sort()`, `FileSpecifier::GetType()`, virtual `found()`.
- **Notes:** 
  - Uses const_iterator for safe traversal
  - Stops immediately if `found()` returns true
  - `dir_entry::operator<` ensures directories appear before files in sorted order

### FindAllFiles::found
- **Signature:** `bool found(FileSpecifier &file)`
- **Purpose:** Callback that appends each found file to the destination vector.
- **Inputs:** `file` ΓÇö the matched file specifier.
- **Outputs/Return:** Always returns `false` to continue searching.
- **Side effects:** Pushes `file` into `dest_vector`.
- **Calls:** `std::vector::push_back()`.
- **Notes:** Simple collector; never terminates search early.

## Control Flow Notes
This file implements the file-discovery phase, likely called during engine initialization to locate game data (maps, resources, scripts). The early-termination design allows both "find first" and "find all" patterns. Blocking operation; not designed for real-time use.

## External Dependencies
- **Includes:** `cseries.h` (STL, types), `FileHandler.h` (`FileSpecifier`, `DirectorySpecifier`, `dir_entry`), `find_files.h` (class declarations)
- **Uses:** `std::vector`, `std::sort`, `std::vector::const_iterator`
- **Defined elsewhere:** `DirectorySpecifier::ReadDirectory()`, `FileSpecifier::GetType()`, `WILDCARD_TYPE` constant
