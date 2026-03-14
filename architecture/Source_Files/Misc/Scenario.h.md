# Source_Files/Misc/Scenario.h

## File Purpose
Defines the `Scenario` singleton class for managing scenario metadata and compatibility information in the Aleph One game engine. Handles parsing of scenario XML elements and stores scenario name, version, ID, and compatible version list.

## Core Responsibilities
- Implement singleton pattern for global scenario state access
- Store and manage scenario metadata (name, version, ID)
- Track compatible scenario versions
- Provide getters and setters with size constraints on string fields
- Support XML element parsing via dedicated parser factory function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Scenario` | class | Singleton container for scenario compatibility metadata and configuration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Scenario::m_instance` | `Scenario*` | static | Singleton instance pointer |

## Key Functions / Methods

### `instance()`
- Signature: `static Scenario *instance()`
- Purpose: Return pointer to singleton instance
- Inputs: None
- Outputs/Return: Static pointer to Scenario singleton
- Side effects: Creates instance on first call (implementation in .cpp)
- Calls: None (visible here)
- Notes: Standard singleton accessor pattern

### `GetName()` / `SetName()`
- Signature: `const string GetName()` / `void SetName(const string name)`
- Purpose: Accessor/mutator for scenario name with truncation
- Inputs: Name string (SetName only)
- Outputs/Return: Name string (GetName) / void (SetName)
- Side effects: SetName truncates name to 32 characters
- Calls: `string::string()` constructor
- Notes: Enforces 32-char max via `string(name, 0, 31)`

### `GetVersion()` / `SetVersion()`
- Signature: `const string GetVersion()` / `void SetVersion(const string version)`
- Purpose: Accessor/mutator for scenario version with truncation
- Inputs: Version string (SetVersion only)
- Outputs/Return: Version string (GetVersion) / void (SetVersion)
- Side effects: SetVersion truncates version to 8 characters
- Calls: `string::string()` constructor
- Notes: Enforces 8-char max via `string(version, 0, 7)`

### `GetID()` / `SetID()`
- Signature: `const string GetID()` / `void SetID(const string id)`
- Purpose: Accessor/mutator for scenario ID with truncation
- Inputs: ID string (SetID only)
- Outputs/Return: ID string (GetID) / void (SetID)
- Side effects: SetID truncates ID to 24 characters
- Calls: `string::string()` constructor
- Notes: Enforces 24-char max via `string(id, 0, 23)`

### `IsCompatible()` / `AddCompatible()`
- Signature: `bool IsCompatible(const string)` / `void AddCompatible(const string)`
- Purpose: Query and register scenario version compatibility
- Inputs: Version string to check or register
- Outputs/Return: Boolean compatibility status (IsCompatible) / void (AddCompatible)
- Side effects: AddCompatible appends to `m_compatibleVersions` vector
- Calls: Not visible (implementation in .cpp)
- Notes: IsCompatible implementation details not shown; likely searches m_compatibleVersions

### `Scenario_GetParser()`
- Signature: `XML_ElementParser *Scenario_GetParser()`
- Purpose: Factory function returning XML parser for scenario elements
- Inputs: None
- Outputs/Return: Pointer to XML_ElementParser (or subclass)
- Side effects: Likely allocates new parser object
- Calls: Not visible (implementation in .cpp)
- Notes: Implementation expected in corresponding .cpp file; parser name is "scenario"

## Control Flow Notes
This class serves a data/configuration role:
1. **Init phase**: Singleton instance created, likely populated from XML parsing
2. **Runtime query**: Game code calls GetName/Version/ID and IsCompatible to validate scenarios
3. **Parser integration**: Scenario_GetParser() bridges this class to the XML parsing framework (inherited from XML_ElementParser)

The private constructor prevents external instantiation, enforcing singleton pattern.

## External Dependencies
- `XML_ElementParser.h` ΓÇö base class for XML element parsing (expected parent class for parser returned by Scenario_GetParser)
- `<string>` ΓÇö STL string for metadata storage
- `<vector>` ΓÇö STL vector for compatible version list
