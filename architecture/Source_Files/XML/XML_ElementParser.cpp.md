# Source_Files/XML/XML_ElementParser.cpp

## File Purpose
Implementation of the `XML_ElementParser` base class, which provides utilities for parsing XML elements, attributes, and text content. Handles numerical and boolean value parsing, element hierarchy management, and UTF-8 string conversion with Mac Roman encoding support.

## Core Responsibilities
- Parse typed values (int16, uint16, int32, uint32, float, boolean) from XML attribute strings with validation
- Manage a hierarchy of child XML element parsers with duplicate prevention
- Perform case-insensitive string matching for XML element and attribute names
- Convert UTF-8 encoded strings to ASCII/Mac Roman for compatibility
- Generate and track descriptive error messages for parsing failures
- Support bounded numerical parsing with min/max range validation

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ElementParser` | class | Base class for parsing individual XML elements; manages children and error state |
| `vector<XML_ElementParser *>` | STL container | Stores child element parsers in hierarchy |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `InitialErrorString` | `static char[]` | static | Default error message on construction |
| `UnrecognizedTagString` | `static char[]` | static | Error: unrecognized XML tag encountered |
| `AttribsMissingString` | `static char[]` | static | Error: required attributes not provided |
| `BadNumericalValueString` | `static char[]` | static | Error: invalid numerical format in attribute |
| `OutOfRangeString` | `static char[]` | static | Error: numerical value outside bounds |
| `BadBooleanValueString` | `static char[]` | static | Error: invalid boolean value format |

## Key Functions / Methods

### ReadInt16Value, ReadUInt16Value, ReadInt32Value, ReadUInt32Value, ReadFloatValue
- **Signature:** `bool Read{Type}Value(const char *String, {type}& Value)`
- **Purpose:** Convenience wrappers to parse typed numerical values from XML attribute strings
- **Inputs:** String to parse, reference to output variable
- **Outputs/Return:** `true` if successfully parsed, `false` otherwise; output parameter contains parsed value
- **Side effects:** Sets `ErrorString` to `BadNumericalValueString` on parse failure
- **Calls:** `ReadNumericalValue()` template with appropriate `sscanf` format string
- **Notes:** The bounded variants also validate `Value >= MinVal && Value <= MaxVal`, setting `OutOfRange` error if violated

### ReadBooleanValueAsInt16, ReadBooleanValueAsUInt16, ReadBooleanValueAsInt32, ReadBooleanValueAsUInt32, ReadBooleanValueAsBool
- **Signature:** `bool ReadBooleanValueAs{Type}(const char *String, {type}& Value)`
- **Purpose:** Parse boolean XML attributes and cast result to target integer/bool type
- **Inputs:** String to parse, reference to output variable
- **Outputs/Return:** `true` if valid boolean, `false` otherwise
- **Side effects:** Sets `ErrorString` to `BadBooleanValueString` on failure
- **Calls:** `ReadBooleanValue()` template, which calls `XML_GetBooleanValue()`

### XML_GetBooleanValue
- **Signature:** `bool XML_GetBooleanValue(const char *String, bool &Value)`
- **Purpose:** Parse string to boolean, accepting multiple formats
- **Inputs:** String (case-insensitive: "1"/"t"/"true" ΓåÆ true; "0"/"f"/"false" ΓåÆ false)
- **Outputs/Return:** `true` if valid format, `false` otherwise; sets `Value` output parameter
- **Side effects:** None (free function, global scope)

### NameMatch
- **Signature:** `bool NameMatch(const char *_Name)`
- **Purpose:** Check if given name matches this element's name (case-insensitive)
- **Inputs:** Name to compare
- **Outputs/Return:** `true` if names match (case-insensitive), `false` otherwise
- **Calls:** `StringsEqual()`

### XML_ElementParser (constructor)
- **Signature:** `XML_ElementParser(const char *_Name)`
- **Purpose:** Initialize a named XML element parser
- **Inputs:** Element name string
- **Outputs/Return:** None
- **Side effects:** Allocates heap memory for name copy; initializes `ErrorString` to `InitialErrorString`
- **Notes:** Name is stored as dynamically allocated C string

### ~XML_ElementParser (destructor)
- **Signature:** `~XML_ElementParser()`
- **Purpose:** Clean up dynamically allocated name
- **Side effects:** Deallocates `Name` via `delete[]`
- **Notes:** Virtual (defined in header), allowing subclass cleanup

### AddChild
- **Signature:** `void AddChild(XML_ElementParser *Child)`
- **Purpose:** Register a child element parser with duplicate prevention
- **Inputs:** Pointer to child parser
- **Side effects:** Appends to `Children` vector if no child with same name already exists
- **Calls:** `GetName()`, `NameMatch()` on each existing child
- **Notes:** O(n) linear search for duplicates; early exit if match found

### ResetChildrenValues
- **Signature:** `void ResetChildrenValues()`
- **Purpose:** Recursively reset parsing state across entire element tree
- **Side effects:** Calls `ResetValues()` and `ResetChildrenValues()` on each child (depth-first)
- **Notes:** Allows re-parsing of same element hierarchy without reinitializing

### FindChild
- **Signature:** `XML_ElementParser *FindChild(const char *_Name)`
- **Purpose:** Locate child element parser by name
- **Inputs:** Child name to search for
- **Outputs/Return:** Pointer to matching child, or `NULL` if not found
- **Calls:** `NameMatch()` on each child
- **Notes:** Linear O(n) search; first match returned

### StringsEqual
- **Signature:** `bool StringsEqual(const char *String1, const char *String2, int MaxStrLen = 32)`
- **Purpose:** Case-insensitive string comparison up to a maximum length
- **Inputs:** Two strings, max comparison length (default 32)
- **Outputs/Return:** `true` if strings match (case-insensitive) within length, `false` otherwise
- **Notes:** Converts both characters to uppercase before comparison; terminates early if either string ends or characters differ; default length suitable for XML element/attribute names

### DeUTF8
- **Signature:** `size_t DeUTF8(const char *InString, size_t InLen, char *OutString, size_t OutMaxLen)`
- **Purpose:** Convert UTF-8 encoded string to ASCII/Mac Roman with fallback for unmappable characters
- **Inputs:** UTF-8 input string, input length, output buffer, output buffer capacity
- **Outputs/Return:** Number of output characters written
- **Side effects:** Writes to `OutString` (does not null-terminate)
- **Calls:** `unicode_to_mac_roman()` (defined elsewhere) for character conversion
- **Notes:** Implements RFC 3629 UTF-8 decoding; invalid/incomplete sequences ΓåÆ '$'; supports 1ΓÇô6 byte sequences; output capped at `OutMaxLen`

### DeUTF8_Pas, DeUTF8_C
- **Signature:** `size_t DeUTF8_Pas(const char *InString, size_t InLen, unsigned char *OutString, size_t OutMaxLen)` and `size_t DeUTF8_C(const char *InString, size_t InLen, char *OutString, size_t OutMaxLen)`
- **Purpose:** Convenience wrappers for UTF-8 conversion to Pascal (length-prefixed) or C (null-terminated) strings
- **Inputs:** UTF-8 input, lengths, output buffer, capacity
- **Outputs/Return:** Character count written (excluding terminator/length byte)
- **Side effects:** DeUTF8_Pas writes length prefix at `OutString[0]` and string starting at `OutString[1]`; DeUTF8_C writes null terminator at `OutString[Len]`
- **Calls:** `DeUTF8()`

## Control Flow Notes
This is a base class providing element-level XML parsing utilities. Subclasses override virtual methods (`Start()`, `End()`, `HandleAttribute()`, `AttributesDone()`, `HandleString()`, `ResetValues()`) to implement domain-specific parsing. The hierarchy allows recursive parsing of nested XML structures via `Parent` pointer and `Children` vector. Error state is tracked in `ErrorString` and can be queried after method calls return `false`.

## External Dependencies
- **cseries.h** ΓÇö engine umbrella header (includes standard types, SDL, MacOS emulation)
- **cstypes.h** ΓÇö fixed-width integer types (int16, uint16, int32, uint32, uint8)
- **XML_ElementParser.h** ΓÇö class declaration, template definitions
- **\<string.h\>** ΓÇö strlen, strcpy
- **\<ctype.h\>** ΓÇö toupper
- **\<vector\>** ΓÇö STL container for children list
- **\<stdio.h\>** ΓÇö sscanf for numerical parsing
- **unicode_to_mac_roman()** ΓÇö external function (defined elsewhere) for character conversion
