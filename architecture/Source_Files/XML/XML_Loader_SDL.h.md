# Source_Files/XML/XML_Loader_SDL.h

## File Purpose
SDL-based XML parser for the Marathon/Aleph One game engine. Extends the generic `XML_Configure` class to load and parse XML configuration files from disk, with file-aware error reporting.

## Core Responsibilities
- Load XML file data from disk via `FileSpecifier` objects
- Scan directories for XML files to parse
- Provide parsed data buffers to the parent `XML_Configure` parser
- Report parsing/reading errors with source filename context
- Manage memory for loaded XML data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_Loader_SDL | class | Concrete XML parser implementation using SDL file I/O |
| FileSpecifier | forward-declared class | Represents file/directory references (defined elsewhere) |

## Global / File-Static State
None.

## Key Functions / Methods

### ParseFile
- **Signature:** `bool ParseFile(FileSpecifier &file)`
- **Purpose:** Load and parse a single XML file
- **Inputs:** FileSpecifier reference to target file
- **Outputs/Return:** bool (success/failure)
- **Side effects:** Allocates `data` buffer, populates `FileName`, invokes parent's `DoParse()`
- **Calls:** FileSpecifier operations (not visible in this file)
- **Notes:** Entry point for parsing individual files

### ParseDirectory
- **Signature:** `bool ParseDirectory(FileSpecifier &dir)`
- **Purpose:** Discover and parse all XML files in a directory
- **Inputs:** FileSpecifier reference to target directory
- **Outputs/Return:** bool (success/failure)
- **Side effects:** Iterates directory contents, calls ParseFile for each XML
- **Calls:** FileSpecifier directory enumeration (not visible in this file)
- **Notes:** Likely filters for .xml or similar extensions

### Virtual Overrides (GetData, ReportReadError, ReportParseError, ReportInterpretError, RequestAbort)
- **Purpose:** Implement abstract callbacks defined by `XML_Configure` base class
- **GetData:** Supplies buffered XML data to expat parser; manages `Buffer`, `BufLen`, `LastOne` fields
- **ReportReadError/ReportParseError/ReportInterpretError:** Format errors with `FileName` for user-facing diagnostics
- **RequestAbort:** May signal early termination if too many errors accumulate

## Control Flow Notes
Initialization ΓåÆ ParseFile/ParseDirectory ΓåÆ GetData (called repeatedly by DoParse) ΓåÆ ReportXxxError (on failures) ΓåÆ RequestAbort (decision to stop or continue).

## External Dependencies
- `XML_Configure.h` (base class providing DoParse, parser lifecycle, expat integration)
- `XML_Configure::Buffer`, `BufLen`, `LastOne` (inherited protected fields filled by GetData)
- `FileSpecifier` (file system abstraction, defined elsewhere)
- expat parser (included via XML_Configure.h)
