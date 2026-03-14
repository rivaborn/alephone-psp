# Source_Files/XML/XML_Loader_SDL.cpp

## File Purpose
SDL-based XML file parser for the Aleph One game engine. Loads and parses XML configuration files from disk, manages file I/O, and reports parsing errors with context (filename, line number, error count limits).

## Core Responsibilities
- Load XML file data into memory buffers and expose to parser
- Parse individual XML files via `ParseFile()`
- Batch-parse all XML files in a directory via `ParseDirectory()`, filtering backup files (~) and Lua scripts (.lua)
- Report read errors, parse errors, and interpretation errors with context
- Enforce error reporting limits and request parse abort if threshold exceeded
- Track current filename for error messages

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `dir_entry` | struct (from FileHandler.h) | Directory entry with name, size, is_directory, is_volume flags |
| `FileSpecifier` | class (from FileHandler.h) | Abstraction for file paths and directory operations |
| `OpenedFile` | class (from FileHandler.h) | RAII wrapper for opened file handles |
| `XML_Configure` | class (base) | Defines parsing interface; actual parsing logic inherited from this |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `MaxErrorsToShow` | const int | static | Threshold (7) for limiting interpretation error output |
| `data` | char* | member | Buffer holding XML file contents (allocated per file) |
| `data_size` | long | member | Size of current file buffer |
| `FileName[256]` | char[] | member | Current filename for error messages (LP: added Oct 24, 2001) |

## Key Functions / Methods

### GetData()
- Signature: `bool GetData()`
- Purpose: Provides loaded XML data to the parser (called during parsing phase)
- Inputs: None (uses member `data` and `data_size`)
- Outputs/Return: `true` if data was successfully loaded, `false` if `data == NULL`
- Side effects: Sets parser members `Buffer`, `BufLen`, `LastOne`
- Calls: None visible
- Notes: Returns early if no data available; assumes `data` is already populated by caller

### ReportReadError()
- Signature: `void ReportReadError()`
- Purpose: Report file I/O failure and abort
- Inputs: None
- Outputs/Return: None (calls `exit(1)`)
- Side effects: Prints to stderr, terminates process
- Calls: `fprintf(stderr, ...)`
- Notes: Non-recoverable; exits immediately

### ReportParseError(const char *ErrorString, int LineNumber)
- Signature: `void ReportParseError(const char *ErrorString, int LineNumber)`
- Purpose: Report XML parse errors with context (filename, line number)
- Inputs: Error message string, line number
- Outputs/Return: None
- Side effects: Prints formatted error to stderr
- Calls: `fprintf(stderr, ...)`
- Notes: Error format includes current `FileName` for user convenience

### ReportInterpretError(const char *ErrorString)
- Signature: `void ReportInterpretError(const char *ErrorString)`
- Purpose: Report semantic/interpretation errors during XML processing, with output throttling
- Inputs: Error message string
- Outputs/Return: None
- Side effects: Prints to stderr (up to 7 errors; subsequent errors suppressed)
- Calls: `GetNumInterpretErrors()` (inherited), `fprintf(stderr, ...)`
- Notes: Throttles output by comparing error count to `MaxErrorsToShow`

### RequestAbort()
- Signature: `bool RequestAbort()`
- Purpose: Signal parser to abort if error threshold exceeded
- Inputs: None
- Outputs/Return: `true` if error count ΓëÑ 7, `false` otherwise
- Side effects: None
- Calls: `GetNumInterpretErrors()` (inherited)
- Notes: Called during parsing to break out early on excessive errors

### ParseFile(FileSpecifier &file_name)
- Signature: `bool ParseFile(FileSpecifier &file_name)`
- Purpose: Load, parse, and process a single XML file
- Inputs: `FileSpecifier` reference to target file
- Outputs/Return: `true` if file opened successfully (parse result is reported but not returned)
- Side effects: Allocates/deallocates `data` buffer; updates `FileName`; calls `DoParse()` (inherited)
- Calls: `file_name.Open()`, `file.GetLength()`, `file_name.GetName()`, `file.Read()`, `DoParse()` (inherited), `fprintf(stderr, ...)`
- Notes: Always deallocates buffer on exit; parse errors reported to stderr but don't affect return value

### ParseDirectory(FileSpecifier &dir)
- Signature: `bool ParseDirectory(FileSpecifier &dir)`
- Purpose: Recursively scan directory and parse all valid XML files
- Inputs: `FileSpecifier` reference to directory path
- Outputs/Return: `true` if directory read succeeded, `false` if `ReadDirectory()` failed
- Side effects: Calls `ParseFile()` for each non-filtered entry; I/O side effects per ParseFile
- Calls: `dir.ReadDirectory()`, `sort()` (std), `boost::algorithm::ends_with()`, `ParseFile()`
- Notes: Filters backup files (suffix ~) and Lua scripts (.lua); sorts entries (directories before files, then by name); ignores subdirectories

## Control Flow Notes
Initialization phase (likely at engine startup):
1. Caller invokes `ParseDirectory()` to load all config files
2. For each file: `ParseFile()` opens, reads, and hands data to inherited `DoParse()` (from `XML_Configure`)
3. `DoParse()` calls back to `GetData()` to retrieve buffer, parses, and reports errors via virtual error functions
4. If error count exceeds threshold, `RequestAbort()` signals early termination
5. Memory cleaned up per file; buffer deallocated even on error

## External Dependencies
- **FileHandler.h**: `FileSpecifier`, `OpenedFile` classes for file I/O abstraction
- **XML_Configure.h**: Base class; defines `DoParse()`, `GetNumInterpretErrors()` (inheritance, not shown)
- **cseries.h**: Common game engine headers and macros
- **boost/algorithm/string/predicate.hpp**: `ends_with()` for Lua script filtering
- **stdio.h, vector, algorithm**: Standard C++ libraries
