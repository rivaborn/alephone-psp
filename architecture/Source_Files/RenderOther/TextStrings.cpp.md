# Source_Files/RenderOther/TextStrings.cpp

## File Purpose
Implements a text string management system that replaces MacOS STR# resources. Provides storage and retrieval of string collections indexed by resource ID, supporting both Pascal and C-string formats via XML configuration. Part of the Aleph One game engine.

## Core Responsibilities
- Manage hierarchical string collections (StringSet) organized by resource ID
- Store strings internally as Pascal strings (length byte + data + null terminator) for dual format support
- Provide public API for inserting, retrieving, counting, and deleting strings
- Parse string definitions from XML markup (`<stringset>` and `<string>` elements)
- Dynamically grow internal string arrays when needed
- Maintain linked list of all active string sets in memory

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| StringSet | class (private) | Container for a collection of strings sharing one resource ID; manages dynamic array of string pointers |
| XML_StringSetParser | class | XML parser for `<stringset>` elements; routes to active StringSet and validates index attribute |
| XML_StringParser | class | XML parser for `<string>` elements; extracts index and text content, converts UTF-8, stores in StringSet |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| StringSetRoot | StringSet* | static (file) | Root pointer to linked list of all StringSet objects; null initially |
| StringSetParser | XML_StringSetParser | static (file) | Singleton parser instance for stringset elements |
| StringParser | XML_StringParser | static (file) | Singleton parser instance for string elements |
| IndexNotFound | const char[] | static (file) | Error message constant used by both parsers |

## Key Functions / Methods

### TS_PutString
- **Signature:** `void TS_PutString(short ID, short Index, unsigned char *String)`
- **Purpose:** Insert or replace a Pascal string in the repository
- **Inputs:** ID (resource set identifier), Index (string position in set), String (Pascal string: length byte + data)
- **Outputs/Return:** None
- **Side effects:** Creates new StringSet if ID not found; reallocates StringSet's array if Index exceeds capacity
- **Calls:** `FindStringSet()`, `StringSet::Add()`

### TS_PutCString
- **Signature:** `void TS_PutCString(short ID, short Index, const char *String)`
- **Purpose:** Insert or replace a C-string (null-terminated) by converting to Pascal format
- **Inputs:** ID (resource set), Index (position), String (C-string, max 255 chars)
- **Outputs/Return:** None
- **Side effects:** Allocates temporary Pascal buffer; calls `StringSet::Add()` which copies data
- **Calls:** `FindStringSet()`, `strlen()`, `memcpy()`, `StringSet::Add()`
- **Notes:** Truncates strings longer than 255 bytes; length byte is written as `(char)theStringLength`

### TS_GetString
- **Signature:** `unsigned char *TS_GetString(short ID, size_t Index)`
- **Purpose:** Retrieve a Pascal string; returns null if not found
- **Inputs:** ID, Index
- **Outputs/Return:** Pointer to Pascal string (length byte + data + null), or NULL
- **Side effects:** None
- **Calls:** Linear search through StringSetRoot linked list

### TS_GetCString
- **Signature:** `char *TS_GetCString(short ID, size_t Index)`
- **Purpose:** Retrieve a string as a C-string (skip the length byte)
- **Inputs:** ID, Index
- **Outputs/Return:** Pointer to first character (String+1 offset), or NULL
- **Side effects:** None
- **Calls:** `TS_GetString()`

### StringSet::Add
- **Signature:** `void StringSet::Add(size_t Index, unsigned char *String)`
- **Purpose:** Store a Pascal string at given index, growing array if needed
- **Inputs:** Index (target position), String (Pascal string)
- **Outputs/Return:** None
- **Side effects:** Allocates/reallocates Strings array; deletes old string at Index if present; copies incoming string
- **Calls:** `objlist_clear()`, `objlist_copy()`, `memcpy()`, `new`, `delete[]`
- **Notes:** Doubles array size repeatedly until Index fits; copies input string (length+1 + content byte) then appends null terminator

### FindStringSet
- **Signature:** `static StringSet *FindStringSet(short ID)`
- **Purpose:** Locate or create StringSet with given ID
- **Inputs:** ID
- **Outputs/Return:** Pointer to StringSet (existing or newly created)
- **Side effects:** Creates and links new StringSet if not found; modifies StringSetRoot on first insertion
- **Calls:** `new StringSet()`
- **Notes:** Walks linked list; appends to end if missing

### TS_DeleteString / TS_DeleteStringSet / TS_DeleteAllStrings
- Linear search and unlink/delete operations; see notes below

## Control Flow Notes

**Initialization:** Parsers are static singletons; created on first `TS_GetParser()` call.

**XML Parsing Flow:**
1. Root element `<stringset index="N">` triggers `XML_StringSetParser::Start()` ΓåÆ finds/creates StringSet with ID
2. Child `<string index="M">` triggers `XML_StringParser::Start()` 
3. Text content decoded via `DeUTF8_Pas()` ΓåÆ `XML_StringParser::HandleString()` ΓåÆ `StringSet::Add()`
4. Empty `<string>` elements load as empty string in `End()` method

**Runtime API:** Public functions search the StringSetRoot linked list. No deferred cleanup; all operations are synchronous.

## External Dependencies

- `<string.h>` ΓÇö `memcpy()`, `strlen()`
- `"cseries.h"` ΓÇö `objlist_clear()`, `objlist_copy()`, cross-platform macros
- `"XML_ElementParser.h"` ΓÇö Base class `XML_ElementParser`, helper functions `StringsEqual()`, `ReadInt16Value()`, `DeUTF8_Pas()`
- Assumed elsewhere: `Str255` typedef (MacOS Pascal string type, `unsigned char[256]`)
