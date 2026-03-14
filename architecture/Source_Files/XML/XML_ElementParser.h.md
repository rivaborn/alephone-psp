# Source_Files/XML/XML_ElementParser.h

## File Purpose
Base class framework for hierarchical XML element parsing. Provides parsing infrastructure for attributes, child elements, and text content. Designed to be subclassed for specific XML element types with custom parsing logic.

## Core Responsibilities
- Manage XML element hierarchy (parent/child relationships)
- Parse and validate numerical values (with optional bounds checking)
- Parse boolean and floating-point values from strings
- Provide lifecycle hooks (Start, End, AttributesDone, HandleString)
- Track parsing errors via ErrorString
- Manage child element registry with case-insensitive lookup
- Support UTF-8 to ASCII string conversion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ElementParser` | class | Base class for parsing individual XML elements |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `XML_GetBooleanValue` | function (extern) | extern | Parse boolean string values ("true"/"false") |

## Key Functions / Methods

### Constructor
- Signature: `XML_ElementParser(const char *_Name)`
- Purpose: Initialize parser with element name
- Inputs: Element tag name
- Side effects: Allocates Name, initializes Children vector and Parent pointer
- Notes: Virtual destructor ensures proper cleanup in derived classes

### Start / End
- Signature: `virtual bool Start()`, `virtual bool End()`
- Purpose: Parsing lifecycle hooksΓÇöcalled when element parsing begins/completes
- Returns: true on success; false sets ErrorString in subclass
- Notes: Default implementation returns true; override in subclasses

### HandleAttribute / AttributesDone
- Signature: `virtual bool HandleAttribute(const char *Tag, const char *Value)`, `virtual bool AttributesDone()`
- Purpose: Process individual attributes, then finalize attribute parsing
- Inputs: Tag=attribute name, Value=attribute string
- Returns: false on validation error
- Side effects: Subclass updates internal state based on attributes

### HandleString
- Signature: `virtual bool HandleString(const char *String, int Length)`
- Purpose: Handle text content embedded within the element
- Inputs: String pointer, content length
- Returns: false on parsing error

### ResetValues / ResetChildrenValues
- Signature: `virtual bool ResetValues()`, `void ResetChildrenValues()`
- Purpose: Restore parser state to initial values; recursive for all children
- Side effects: Clears all parsed state

### ReadNumericalValue (template)
- Signature: `template<class T> bool ReadNumericalValue(const char *String, const char *Format, T& Value)`
- Purpose: Parse typed numerical value using sscanf-style format string
- Inputs: String to parse, scanf format, reference to output value
- Outputs: Value filled with parsed result
- Returns: true if exactly 1 value parsed; false calls BadNumericalValue()
- Calls: sscanf, BadNumericalValue()

### ReadBoundedNumericalValue (template)
- Signature: `template<class T> bool ReadBoundedNumericalValue(const char *String, const char *Format, T& Value, const T& MinVal, const T& MaxVal)`
- Purpose: Parse numerical value and enforce min/max bounds
- Returns: false if out of range; calls OutOfRange()

### ReadBooleanValue / ReadFloatValue
- Purpose: Specialized numeric parsing for booleans and floats
- Convenience methods: ReadInt16Value, ReadUInt32Value, ReadBooleanValueAsInt32, etc. (wrapper methods to reduce code bloat)

### AddChild / FindChild
- Signature: `void AddChild(XML_ElementParser *Child)`, `XML_ElementParser *FindChild(const char *_Name)`
- Purpose: Register/lookup child parsers by name
- Notes: FindChild returns NULL if not found; prevents duplicate child names

### NameMatch / GetName
- Signature: `bool NameMatch(const char *_Name)`, `char *GetName()`
- Purpose: Case-insensitive name matching and name getter
- Calls: StringsEqual (case-insensitive comparison)

## Control Flow Notes
Hierarchical parsing sequence:
1. Element creation ΓåÆ `Start()` called
2. Attributes iterated ΓåÆ `HandleAttribute()` per attribute, then `AttributesDone()`
3. Text content ΓåÆ `HandleString()` called
4. Child elements processed recursively (each has own Start/End cycle)
5. Element completion ΓåÆ `End()` called

Parent pointer enables upward navigation during parsing. Error handling is callback-based (subclass implements error notification).

## External Dependencies
- `<vector>` (STL) ΓÇö child element container
- `<stdio.h>` ΓÇö sscanf for numerical parsing
- `cstypes.h` ΓÇö cross-platform integer type definitions (int16, uint16, int32, uint32)
- `XML_GetBooleanValue()` ΓÇö extern function for boolean string parsing
- `StringsEqual()` ΓÇö case-insensitive string comparison (defined in this file)
- `DeUTF8()`, `DeUTF8_Pas()`, `DeUTF8_C()` ΓÇö UTF-8 conversion utilities (defined in this file)

## Notes
- UTF-8 support with ASCII conversion (De-UTF8 functions)
- All string comparisons are case-insensitive (element names, attribute names)
- Data members: `Name` (char*), `Children` (vector), `Parent` (pointer), `ErrorString` (const char*)
- Error reporting via virtual error methods: UnrecognizedTag(), AttribsMissing(), BadNumericalValue(), OutOfRange(), BadBooleanValue()
