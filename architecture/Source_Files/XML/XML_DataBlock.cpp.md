# Source_Files/XML/XML_DataBlock.cpp

## File Purpose
Implements the XML_DataBlock class for parsing XML contained in memory buffers. Handles error reporting at three levels (read, parse, interpretation) with platform-specific error dialogs and graceful logging. Part of the Aleph One game engine's XML configuration system.

## Core Responsibilities
- Provides XML data blocks to the parser via GetData()
- Reports fatal read errors with platform-specific error dialogs (Mac/SDL)
- Reports XML parsing errors with line numbers for debugging
- Reports interpretation errors via logging with throttling to avoid spam
- Requests parsing abortion when error threshold is exceeded
- Maintains source name for debugging context

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MaxErrorsToShow | const int | static | Limit interpretation errors displayed (set to 7); prevents excessive error spam during parsing |

## Key Functions / Methods

### GetData
- Signature: `bool XML_DataBlock::GetData()`
- Purpose: Provides XML data block to the parser
- Inputs: Uses member `Buffer` and `BufLen` (set via `ParseData()`)
- Outputs/Return: `bool` (always `true`)
- Side effects: Sets `LastOne = true` to mark this as final data block
- Calls: None
- Notes: Simple validation wrapper; asserts Buffer and BufLen are valid. Single-block designΓÇöalways marks data as last block.

### ReportReadError
- Signature: `void XML_DataBlock::ReportReadError()`
- Purpose: Report fatal error when reading/loading data from source
- Inputs: Uses `SourceName` member for context
- Outputs/Return: void
- Side effects: **Terminates program**: Mac shows fatal dialog then `ExitToShell()`; SDL prints to stderr then `exit(1)`
- Calls: `csprintf`/`psprintf` (formatting), `SimpleAlert`/`Alert` (Mac dialogs), `ParamText` (Mac), `ExitToShell`, `fprintf`, `exit`
- Notes: Fatal error path; platform-specific error UI (Carbon vs pre-Carbon Mac; SDL). Uses `SourceName` or `"[]"` as fallback.

### ReportParseError
- Signature: `void XML_DataBlock::ReportParseError(const char *ErrorString, int LineNumber)`
- Purpose: Report XML parse error with line number context for debugging
- Inputs: `ErrorString` (error message), `LineNumber` (location in XML)
- Outputs/Return: void
- Side effects: **Terminates on Mac**: shows dialog then `ExitToShell()`; SDL only prints to stderr without exit
- Calls: `csprintf`/`psprintf`, `SimpleAlert`/`Alert`, `ParamText`, `ExitToShell`, `fprintf`
- Notes: Inconsistent severity: fatal on Mac, non-fatal log on SDL. Includes line number in output for traceability.

### ReportInterpretError
- Signature: `void XML_DataBlock::ReportInterpretError(const char *ErrorString)`
- Purpose: Report non-fatal XML interpretation error with throttling
- Inputs: `ErrorString` (error message)
- Outputs/Return: void
- Side effects: Calls `logAnomaly1()` or `logAnomaly()` via logging system; increments internal error count
- Calls: `GetNumInterpretErrors()` (inherited), `logAnomaly1`, `logAnomaly`
- Notes: Graceful degradationΓÇöshows first 7 errors, then single `"(more errors not shown)"` message on 8th+. Avoids log spam.

### RequestAbort
- Signature: `bool XML_DataBlock::RequestAbort()`
- Purpose: Check if parsing should abort due to accumulated errors
- Inputs: None
- Outputs/Return: `bool`ΓÇö`true` if `GetNumInterpretErrors() >= MaxErrorsToShow`
- Side effects: None
- Calls: `GetNumInterpretErrors()` (inherited from `XML_Configure`)
- Notes: Called by parser to halt gracefully when error threshold exceeded; allows cleanup vs. fatal crash.

## Control Flow Notes
Part of XML data loading/initialization:
1. User calls `ParseData(buffer, length)` to set up data block
2. Calls inherited `DoParse()` which invokes `GetData()` to fetch buffer
3. Parser calls `ReportReadError()` if data load fails (fatal) or `ReportParseError()` for XML syntax errors (fatal on Mac)
4. Non-fatal interpretation errors flow through `ReportInterpretError()` (logged, throttled)
5. Parser checks `RequestAbort()` to halt early if error count >= 7
6. On Mac, fatal errors terminate immediately; on SDL, some errors only log

## External Dependencies
- **cseries.h**: Cross-platform utilities (`csprintf`, `psprintf`, `SimpleAlert`, `ParamText`, `Alert`, `ExitToShell`)
- **XML_DataBlock.h**: Class declaration; parent class `XML_Configure`
- **Logging.h**: Logging system (`logAnomaly1`, `logAnomaly`, `GetCurrentLogger`)
- **stdio.h (implicit)**: `fprintf` for SDL stderr output
- **stdlib.h (implicit)**: `exit()` for SDL termination
- Inherited: `GetNumInterpretErrors()`, `DoParse()` from `XML_Configure`
