# Source_Files/RenderOther/TextStrings.h

## File Purpose
Header file declaring a text-string repository interface for the Aleph One game engine. Replaces macOS STR# resource management with a unified API for storing, retrieving, and managing text strings organized by resource ID and index. Supports both Pascal-format (length-prefixed) and C-style (null-terminated) strings.

## Core Responsibilities
- Store and retrieve strings by resource ID and index
- Convert between Pascal-format and C-style string representations
- Query string set presence and string count
- Delete individual strings or entire string sets
- Provide XML parsing interface for string configuration
- Manage in-memory string repository

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_ElementParser | class (fwd decl) | Parser for reading/writing string sets from XML configuration |

## Global / File-Static State
None.

## Key Functions / Methods

### TS_PutString
- Signature: `void TS_PutString(short ID, short Index, unsigned char *String)`
- Purpose: Store a Pascal-format string in the repository
- Inputs: Resource ID, index within that ID's set, pointer to Pascal string (length byte + chars + null)
- Outputs/Return: None
- Side effects: Allocates/replaces string in memory; overwrites existing string at same ID+Index
- Notes: String format is `[length][characters][null]`; allows same ID+Index to be updated

### TS_PutCString
- Signature: `void TS_PutCString(short ID, short Index, const char *String)`
- Purpose: Store a C-style null-terminated string, converting to Pascal format internally
- Inputs: Resource ID, index, pointer to C string
- Outputs/Return: None
- Side effects: Allocates/replaces string in memory
- Notes: Convenience wrapper around TS_PutString

### TS_GetString
- Signature: `unsigned char *TS_GetString(short ID, size_t Index)`
- Purpose: Retrieve a Pascal-format string
- Inputs: Resource ID, index
- Outputs/Return: Pointer to Pascal string, or NULL if not found
- Side effects: None
- Notes: Returned pointer points to internal storage; string is usable as both Pascal and C format

### TS_GetCString
- Signature: `char *TS_GetCString(short ID, size_t Index)`
- Purpose: Retrieve a C-style string
- Inputs: Resource ID, index
- Outputs/Return: Pointer to C string (char form of same data as TS_GetString)
- Outputs/Return: Pointer, or NULL if not found

### TS_IsPresent
- Signature: `bool TS_IsPresent(short ID)`
- Purpose: Check if a string set with the given ID exists
- Inputs: Resource ID
- Outputs/Return: True if ID has any strings

### TS_CountStrings
- Signature: `size_t TS_CountStrings(short ID)`
- Purpose: Count contiguous strings in a set, starting from index 0
- Inputs: Resource ID
- Outputs/Return: Count of strings

### TS_DeleteString
- Signature: `void TS_DeleteString(short ID, short Index)`
- Purpose: Remove a single string from the repository
- Inputs: Resource ID, index
- Outputs/Return: None
- Side effects: Deallocates string; may leave gap in set

### TS_DeleteStringSet
- Signature: `void TS_DeleteStringSet(short ID)`
- Purpose: Remove all strings with a given ID
- Inputs: Resource ID
- Outputs/Return: None
- Side effects: Deallocates entire set

### TS_DeleteAllStrings
- Signature: `void TS_DeleteAllStrings()`
- Purpose: Clear entire string repository
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates all strings

### TS_GetParser
- Signature: `XML_ElementParser *TS_GetParser()`
- Purpose: Obtain parser for reading/writing string configuration from XML
- Inputs: None
- Outputs/Return: Pointer to reusable XML_ElementParser named "stringset"
- Side effects: None (parser is managed internally; caller must not delete)
- Notes: Parser returned is singleton; should not be freed by caller

## Control Flow Notes
Not inferable from headerΓÇöthis is a pure interface declaration. Likely called during game initialization (loading from XML) and during gameplay (UI/HUD text lookup).

## External Dependencies
- `XML_ElementParser` ΓÇô forward-declared; defined elsewhere; used for XML string configuration
