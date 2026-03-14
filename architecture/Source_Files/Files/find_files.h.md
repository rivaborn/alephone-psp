# Source_Files/Files/find_files.h

## File Purpose
Header file defining cross-platform file-finding abstractions for macOS (classic Mac API) and SDL-based platforms. Enables searching for files by typecode with support for recursive directory traversal, filtering via callbacks, and directory-change notifications.

## Core Responsibilities
- Define platform-specific `FileFinder` class hierarchy for file discovery
- Support file enumeration by typecode with optional recursion
- Provide callback mechanisms for filtering and processing found files
- Abstract differences between macOS Classic/Carbon APIs and SDL file operations
- Define `WILDCARD_TYPE` constant for unrestricted type matching

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `FileFinder` (macOS) | class | Encapsulates file search state and control; performs enumeration using CInfoPBRec |
| `FileFinder` (SDL) | class | Abstract base for file-search implementations with platform-independent Find() interface |
| `FindAllFiles` (SDL) | class | Concrete `FileFinder` subclass that collects found files into a vector |
| `WILDCARD_TYPE` | const Typecode | Symbolic constant (_typecode_unknown) for matching any file type |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `WILDCARD_TYPE` | const Typecode | global | Default file type matching any typecode during searches |

## Key Functions / Methods

### `FileFinder::Find()` (macOS version)
- Signature: `bool Find()`
- Purpose: Execute file search according to configured parameters (BaseDir, Type, flags, callbacks)
- Inputs: Configured via public member variables (BaseDir, Type, flags, buffer, max, callback, user_data, directory_change_callback)
- Outputs/Return: `bool` (success); found count in `count` member; matches in `buffer` (if provided)
- Side effects: Modifies `buffer` contents; may call `callback` and `directory_change_callback` functions with file specs; sets `Err` on failure
- Calls: `Enumerate()` (private); `callback()` (user-supplied); `directory_change_callback()` (user-supplied)
- Notes: Recurses into subdirectories if `_ff_recurse` flag set; `callback` returns true to include file, false to skip; if `_callback_only` flag and callback returns false, terminates enumeration

### `FileFinder::Find()` (SDL version)
- Signature: `bool Find(DirectorySpecifier &dir, Typecode type, bool recursive = true)`
- Purpose: Search directory for files of given type; invoke `found()` for each match
- Inputs: `dir` (search root), `type` (file typecode), `recursive` (enable subdirectory traversal, default true)
- Outputs/Return: `bool` (success); side effects call `found()` for matches
- Side effects: Calls `found()` virtual method for each matching file; may recursively enter subdirectories
- Calls: `found()` (virtual, implemented by subclass)
- Notes: Pure virtual base class; caller must subclass and override `found()`

### `FileFinder::Enumerate()` (macOS, private)
- Signature: `bool Enumerate(DirectorySpecifier& Dir)`
- Purpose: Recursive helper that performs actual enumeration using Mac OS API
- Inputs: Directory to enumerate
- Outputs/Return: `bool`
- Side effects: Internal recursion; populates CInfoPBRec state
- Notes: Called by `Find()`

### `FindAllFiles::found()` (SDL)
- Signature: `bool found(FileSpecifier &file)`
- Purpose: Implementation callback that appends found file to destination vector
- Inputs: `file` (FileSpecifier of matched file)
- Outputs/Return: Always returns true (continue search)
- Side effects: Appends to `dest_vector`

### `FindAllFiles::FindAllFiles()` (SDL constructor)
- Signature: `FindAllFiles(vector<FileSpecifier> &v) : dest_vector(v) {dest_vector.clear();}`
- Purpose: Initialize search object with destination vector; clear any prior contents
- Inputs: Reference to vector to populate
- Side effects: Clears the provided vector

## Control Flow Notes
Initialization occurs via constructor and public member setup; `Find()` is the entry point. On macOS, `Find()` calls private `Enumerate()` which recursively traverses directories, invoking callbacks as files are matched. On SDL, `Find()` is implemented by subclasses (e.g., `FindAllFiles`) that override the `found()` virtual method. No explicit shutdown required.

## External Dependencies
- **Includes**: `FileHandler.h` (provides `FileSpecifier`, `DirectorySpecifier`, `Typecode`, `OpenedFile`, `OpenedResourceFile`, `tags.h`)
- **macOS platform**: `Carbon/Carbon.h` (or `Files.h`, `Resources.h`); uses `CInfoPBRec`, `OSType`, `OSErr`, `DirectorySpecifier`
- **SDL platform**: `<vector>`, `<string>`, `SDL.h`; uses `vector<FileSpecifier>`, `std::string`
- **Symbols defined elsewhere**: `FileSpecifier`, `DirectorySpecifier`, `Typecode`, `_typecode_unknown`, `OpenedFile`, `OpenedResourceFile`
