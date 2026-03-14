# Source_Files/XML/XML_DataBlock.h

## File Purpose
Defines `XML_DataBlock`, a concrete parser that loads and parses XML data from memory buffers. It inherits from `XML_Configure` to handle XML parsing for data blocks in the Aleph One engine's configuration system.

## Core Responsibilities
- Provide a data-block-specific XML parsing interface by implementing the `XML_Configure` contract
- Accept XML source data from memory buffers via `ParseData()`
- Optionally track the source location/name of parsed XML for debugging
- Implement error reporting hooks (read, parse, interpretation errors)
- Coordinate with the Expat parser to parse buffered XML content

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_DataBlock` | class | Concrete parser for XML data blocks in memory; inherits from `XML_Configure` |

## Global / File-Static State
None.

## Key Functions / Methods

### ParseData
- **Signature:** `bool ParseData(char *_Buffer, size_t _BufLen)`
- **Purpose:** Entry point to parse XML from a memory buffer.
- **Inputs:** `_Buffer` (pointer to XML character data), `_BufLen` (byte length of buffer)
- **Outputs/Return:** Boolean success/failure
- **Side effects:** Sets member variables `Buffer` and `BufLen`; calls `DoParse()` (inherited)
- **Calls:** `DoParse()` (inherited from `XML_Configure`)
- **Notes:** Simple wrapper that stashes buffer state and delegates to parent class parser.

### Constructor
- **Signature:** `XML_DataBlock()`
- **Purpose:** Initialize a new data block parser instance.
- **Side effects:** Sets `SourceName` to `NULL` and `Buffer` to `NULL`

## Control Flow Notes
Part of the engine's XML configuration initialization pipeline. Called during load-time to parse XML data blocks stored in memory (e.g., from resources or loaded files). The inherited `DoParse()` method iterates by calling `GetData()`, `ReportReadError()`, `ReportParseError()`, etc., which are defined as protected virtual methods here (but implementations are likely in subclasses or the parent).

## External Dependencies
- **`XML_Configure.h`** ΓÇö parent class; defines the core parsing interface and Expat integration
- **`expat.h`** (via `XML_Configure.h`) ΓÇö the Expat XML parser library
- **`XML_ElementParser.h`** (via `XML_Configure.h`) ΓÇö element parsing infrastructure
