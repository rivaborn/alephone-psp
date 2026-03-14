# Source_Files/Misc/PlayerName.cpp

## File Purpose
Manages the player name used in netgames for the Aleph One engine. Provides storage, retrieval, and XML-based configuration parsing for the player's display name, initialized to "Marathon Player" by default.

## Core Responsibilities
- Store and retrieve the player's name as a static buffer
- Parse player name from XML configuration files
- Initialize default player name on parser setup
- Provide XML element parser interface for configuration loading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_SimpleStringParser` | class | Extends `XML_ElementParser` to handle XML string parsing for player name configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `PlayerName` | `unsigned char[256]` | static | Stores the player name in Pascal-string format (length byte + string data) |
| `PlayerNameParser` | `XML_SimpleStringParser` | static | Singleton parser instance for XML-based player name loading |

## Key Functions / Methods

### GetPlayerName()
- Signature: `unsigned char *GetPlayerName()`
- Purpose: Retrieve the currently stored player name
- Inputs: None
- Outputs/Return: Pointer to the `PlayerName` buffer
- Side effects: None
- Calls: None
- Notes: Returns a direct pointer to static storage; caller must not exceed 255-character string length

### XML_SimpleStringParser::HandleString()
- Signature: `bool HandleString(const char *String, int Length)`
- Purpose: Parse a UTF-8 string from XML and store it as the player name
- Inputs: UTF-8 string pointer, string length in bytes
- Outputs/Return: `true` (always succeeds)
- Side effects: Overwrites `PlayerName` buffer via `DeUTF8_Pas()`
- Calls: `DeUTF8_Pas()` (UTF-8 to Pascal-string conversion)
- Notes: Converts UTF-8 to Pascal-string format; truncates to 255 characters

### PlayerName_GetParser()
- Signature: `XML_ElementParser *PlayerName_GetParser()`
- Purpose: Initialize the default player name and return the XML parser
- Inputs: None
- Outputs/Return: Pointer to static `PlayerNameParser` instance
- Side effects: Sets `PlayerName` to "Marathon Player" (14-byte Pascal string)
- Calls: `strlen()`, `memcpy()`
- Notes: Called once during initialization; establishes default name before XML parsing

## Control Flow Notes
**Initialization sequence:**
1. `PlayerName_GetParser()` is called at startup to initialize the default name and obtain the parser
2. XML configuration system invokes `PlayerNameParser` to parse custom name if present
3. `GetPlayerName()` is called at runtime to retrieve the player's name for display in netgames

## External Dependencies
- `cseries.h` ΓÇö Core engine utilities and platform abstractions
- `PlayerName.h` ΓÇö Public interface declarations
- `XML_ElementParser` ΓÇö Base class for XML parsing (defined elsewhere)
- `DeUTF8_Pas()` ΓÇö UTF-8 to Pascal-string conversion utility (defined elsewhere)
- Standard: `<string.h>` (for `strlen()`, `memcpy()`)
