# Source_Files/XML/XML_Configure.cpp

## File Purpose
Implementation of the XML_Configure class, which orchestrates XML file parsing for the Aleph One game engine. Uses the Expat parser library to read XML files and delegates semantic interpretation to a tree of XML_ElementParser objects.

## Core Responsibilities
- Manages Expat parser creation, setup, and lifecycle
- Implements Expat callbacks (static wrappers) that delegate to instance methods
- Navigates and maintains a tree of XML element parsers (CurrentElement)
- Handles XML element lifecycle: start, attribute processing, character data, end
- Accumulates and reports XML parsing and semantic interpretation errors
- Orchestrates the read-parse loop, feeding data chunks to Expat until completion

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_Parser | opaque handle (from Expat) | Represents parser state; used to control parsing and retrieve error info |
| XML_ElementParser | class (external) | Represents a single XML element in a tree; handles attribute/character processing |

## Global / File-Static State
None.

## Key Functions / Methods

### StaticStartElement
- Signature: `static void StaticStartElement(void *UserData, const char *Name, const char **Attributes)`
- Purpose: Expat callback for XML element opening tags; delegates to instance method via UserData
- Inputs: UserData (cast to XML_Configure*), element name, null-terminated attribute array (alternating name/value pairs)
- Outputs/Return: void
- Side effects: Indirect via StartElement (modifies CurrentElement, accumulates errors)
- Calls: StartElement
- Notes: Required by Expat; used to convert Expat's callback model to object-oriented dispatch

### StaticEndElement
- Signature: `static void StaticEndElement(void *UserData, const char *Name)`
- Purpose: Expat callback for XML element closing tags; delegates to instance method
- Inputs: UserData (cast to XML_Configure*), element name
- Outputs/Return: void
- Side effects: Indirect via EndElement (modifies CurrentElement, accumulates errors)
- Calls: EndElement

### StaticCharacterData
- Signature: `static void StaticCharacterData(void *UserData, const char *String, int Length)`
- Purpose: Expat callback for character data between tags; delegates to instance method
- Inputs: UserData (cast to XML_Configure*), character string, byte length
- Outputs/Return: void
- Side effects: Indirect via CharacterData (accumulates errors)
- Calls: CharacterData

### StartElement
- Signature: `void StartElement(const char *Name, const char **Attributes)`
- Purpose: Processes element opening: navigates element tree, validates structure, processes attributes
- Inputs: Element name, null-terminated attribute array (pairs of name/value)
- Outputs/Return: void
- Side effects: Modifies CurrentElement (descends into child), accumulates NumInterpretErrors
- Calls: FindChild, Start, HandleAttribute, AttributesDone, ComposeInterpretError, XML_GetCurrentLineNumber
- Notes: Iterates attributes via pointer arithmetic (AttrPtr++); errors non-fatal (parsing continues); unrecognized child elements are caught as interpretation errors

### EndElement
- Signature: `void EndElement(const char *Name)`
- Purpose: Closes current element, returns to parent, validates element closure
- Inputs: Element name (must match CurrentElement)
- Outputs/Return: void
- Side effects: Modifies CurrentElement (sets to parent), accumulates errors
- Calls: NameMatch, End, ComposeInterpretError, XML_GetCurrentLineNumber
- Notes: Silently returns if name doesn't match (allows graceful handling of unrecognized elements)

### CharacterData
- Signature: `void CharacterData(const char *String, int Length)`
- Purpose: Delegates character data to current element for semantic processing
- Inputs: Character data string, byte length
- Outputs/Return: void
- Side effects: Accumulates errors via ComposeInterpretError
- Calls: HandleString, ComposeInterpretError, GetName, XML_GetCurrentLineNumber

### DoParse
- Signature: `bool DoParse()`
- Purpose: Main parsing orchestration; reads data in chunks and feeds to Expat parser
- Inputs: None (uses virtual GetData() to fetch buffers, Buffer/BufLen/LastOne set by subclass)
- Outputs/Return: bool (true if parse successful and NumInterpretErrors == 0, false on read/parse failure)
- Side effects: Creates Parser via XML_ParserCreate, configures callbacks, parses data, frees Parser; resets/increments NumInterpretErrors
- Calls: XML_ParserCreate, XML_SetUserData, XML_SetElementHandler, XML_SetCharacterDataHandler, GetData, XML_Parse, RequestAbort, XML_ParserFree, ReportReadError, ReportParseError, XML_ErrorString, XML_GetErrorCode, XML_GetCurrentLineNumber
- Notes: Loop continues until LastOne=true; GetData() is virtual (subclass reads file); Expat invokes callbacks during XML_Parse; parse failure or read failure returns false immediately

### ComposeInterpretError
- Signature: `void ComposeInterpretError(const char *Format, ...)`
- Purpose: Formats semantic validation errors (non-XML-syntax) and reports them
- Inputs: printf-style format string and variadic arguments
- Outputs/Return: void
- Side effects: Increments NumInterpretErrors, calls ReportInterpretError (virtual)
- Calls: vsprintf, ReportInterpretError
- Notes: Uses 256-byte stack buffer; intended for schema/validation errors, not parser errors

## Control Flow Notes
**InitializationΓåÆParsingΓåÆShutdown:**
1. Subclass creates XML_Configure instance, sets CurrentElement to a root parser object
2. Subclass calls DoParse()
3. DoParse creates Expat parser, registers callbacks, enters read-parse loop
4. Each iteration: GetData() (subclass reads file chunk), XML_Parse (Expat invokes callbacks)
5. Callbacks navigate element tree (CurrentElement) and validate structure
6. Loop exits when LastOne=true; parser freed; returns success/failure based on error count

**No explicit init/shutdown phases visible**; relies on virtual method overrides (GetData, ReportError variants, RequestAbort) for subclass customization.

## External Dependencies
- **Standard C**: `<stdio.h>`, `<stdarg.h>` (vsprintf, va_list, va_start, va_end)
- **Expat library**: XML_Parser, XML_ParserCreate, XML_SetUserData, XML_SetElementHandler, XML_SetCharacterDataHandler, XML_Parse, XML_ParserFree, XML_ErrorString, XML_GetErrorCode, XML_GetCurrentLineNumber (all imported from "expat.h")
- **Game engine**: "cseries.h" (utilities); "XML_Configure.h" (class definition)
- **XML_ElementParser** (external class): FindChild, Parent, GetName, Start, HandleAttribute, AttributesDone, End, HandleString, NameMatch, ErrorString field (defined elsewhere)
