# Source_Files/XML/XML_Configure.h

## File Purpose
Abstract base class that orchestrates XML parsing for Marathon engine configuration. Wraps the Expat XML parser library and delegates element-level interpretation to a hierarchical `XML_ElementParser` structure. Subclasses provide file I/O and error handling policy.

## Core Responsibilities
- Manage the Expat parser lifecycle and coordinate XML parsing flow
- Provide callback routing from Expat's C API to C++ instance methods
- Maintain a stack of active element parsers for nested XML structures
- Track and report parsing, XML syntax, and interpretation errors
- Define abstract interface for subclasses to supply data and handle errors

## Key Types / Data Structures
| Name | Kind | Purpose |
| --- | --- | --- |
| `XML_Configure` | class | Abstract base for XML configuration parsers |
| `XML_Parser` | typedef (external) | Opaque Expat parser handle |
| `XML_ElementParser` | class (external) | Hierarchical element parser for semantic validation |

## Global / File-Static State
| Name | Type | Scope | Purpose |
| --- | --- | --- | --- |
| `NumInterpretErrors` | `int` | instance | Error counter; limits verbosity of error output |
| `Parser` | `XML_Parser` | instance | Active Expat parser instance |
| `Buffer` | `char*` | instance (protected) | Current XML data chunk to parse |
| `BufLen` | `size_t` | instance (protected) | Byte count of current buffer |
| `LastOne` | `bool` | instance (protected) | Flag: is current buffer the final chunk? |
| `CurrentElement` | `XML_ElementParser*` | instance (public) | Currently active element parser in hierarchy |

## Key Methods

### DoParse
- **Signature**: `bool DoParse()`
- **Purpose**: Main entry point; drives the complete parsing cycle
- **Inputs**: None (uses protected members `Buffer`, `BufLen`, `LastOne` populated by subclass)
- **Outputs/Return**: `bool` ΓÇö `true` if parse completed successfully, `false` on fatal error
- **Side effects**: Repeatedly calls `GetData()`; feeds data to Expat; invokes static callback methods; updates parser state
- **Calls**: Expat API (implementation in `.cpp`); private methods `StartElement()`, `EndElement()`, `CharacterData()`
- **Notes**: Streaming design allows incremental parsing; full loop and error handling in implementation file

### GetData (pure virtual)
- **Signature**: `virtual bool GetData() = 0`
- **Purpose**: Abstract method for subclass to supply next XML data chunk
- **Inputs**: None (subclass must populate protected members)
- **Outputs/Return**: `bool` ΓÇö `true` if data read successfully, `false` on I/O failure
- **Side effects**: Subclass implementation must set `Buffer`, `BufLen`, `LastOne` before returning
- **Calls**: Called by `DoParse()` in a loop until `LastOne == true`
- **Notes**: Enables both file-based and streaming/memory-based parsing strategies

### ComposeInterpretError
- **Signature**: `void ComposeInterpretError(const char *Format, ...)`
- **Purpose**: Format and log a semantic/validation error (printf-style)
- **Inputs**: Format string and variadic arguments
- **Outputs/Return**: None
- **Side effects**: Increments `NumInterpretErrors`; calls `ReportInterpretError()` with formatted message
- **Calls**: `ReportInterpretError()` (implementation file)
- **Notes**: Used to report XML content errors (e.g., invalid attribute values, missing required elements)

### ReportReadError (virtual)
- **Signature**: `virtual void ReportReadError() {}`
- **Purpose**: Called when `GetData()` fails (I/O error)
- **Inputs**: None
- **Outputs/Return**: None
- **Notes**: Default no-op; subclass should override to handle file errors

### ReportParseError (virtual)
- **Signature**: `virtual void ReportParseError(const char *ErrorString, int LineNumber) {}`
- **Purpose**: Called on XML syntax error detected by Expat
- **Inputs**: Error description string; line number in source
- **Outputs/Return**: None
- **Notes**: Default no-op; subclass should override

### ReportInterpretError (virtual)
- **Signature**: `virtual void ReportInterpretError(const char *ErrorString) {}`
- **Purpose**: Called when semantic validation fails (by `ComposeInterpretError()`)
- **Inputs**: Error message
- **Outputs/Return**: None
- **Notes**: Default no-op; subclass should override

### RequestAbort (virtual)
- **Signature**: `virtual bool RequestAbort() {return false;}`
- **Purpose**: Allows subclass to signal early termination (e.g., after too many errors)
- **Inputs**: None
- **Outputs/Return**: `bool` ΓÇö `true` to abort, `false` to continue
- **Notes**: Called during parsing to allow graceful error recovery

### GetNumInterpretErrors
- **Signature**: `int GetNumInterpretErrors() {return NumInterpretErrors;}`
- **Purpose**: Query count of semantic errors encountered
- **Inputs**: None
- **Outputs/Return**: `int` ΓÇö current error count
- **Notes**: Inline getter; no side effects

## Control Flow Notes
This class orchestrates a callback-driven parsing architecture:
1. Subclass calls `DoParse()`
2. `DoParse()` enters loop: calls `GetData()`, feeds buffer to Expat
3. Expat invokes static callback methods (`StaticStartElement`, `StaticEndElement`, `StaticCharacterData`) with `this` pointer in `UserData`
4. Static methods dispatch to instance methods (`StartElement`, `EndElement`, `CharacterData`), which delegate to `CurrentElement` (XML_ElementParser hierarchy)
5. Errors are reported via virtual methods; subclass decides whether to abort

No frame/render cycle involvement. Designed as a one-time configuration initialization step that runs at engine startup.

## External Dependencies
- **expat.h**: Low-level C XML parsing library; provides `XML_Parser` type and callback registration functions
- **XML_ElementParser.h**: Hierarchical element parser; interprets parsed XML structure semantically
- Standard C: `<stdlib.h>`, `<cstdarg>` (inferred for variadic `ComposeInterpretError`)
