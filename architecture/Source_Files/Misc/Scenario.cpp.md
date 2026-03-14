# Source_Files/Misc/Scenario.cpp

## File Purpose
Implements XML parsing and singleton management for scenario metadata in the Aleph One game engine. Handles parsing scenario name, version, ID, and compatibility information from XML configuration files.

## Core Responsibilities
- Manage Scenario singleton instance and metadata (name, version, ID)
- Parse `<scenario>` and `<can_join>` XML elements and attributes
- Maintain and validate a list of compatible scenario versions
- Provide a factory function for creating the scenario XML parser hierarchy

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| Scenario | class | Singleton for managing current scenario metadata and compatibility checks |
| XML_ScenarioParser | class | Parses `<scenario>` element attributes (name, version, id) |
| XML_CanJoinParser | class | Parses `<can_join>` element text content to build compatibility list |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| Scenario::m_instance | Scenario* | static (class member) | Singleton instance |
| ScenarioParser | XML_ScenarioParser | static | Reusable parser for `<scenario>` tags |
| CanJoinParser | XML_CanJoinParser | static | Reusable parser for `<can_join>` tags |

## Key Functions / Methods

### Scenario::instance()
- Signature: `static Scenario *instance()`
- Purpose: Lazy-initialize and return the singleton instance
- Inputs: None
- Outputs/Return: Pointer to the global Scenario instance
- Side effects: Creates Scenario on first call, allocates heap memory (never freed)
- Calls: `new Scenario()`
- Notes: Classic lazy-singleton pattern; no thread safety

### Scenario::AddCompatible()
- Signature: `void AddCompatible(const string Compatible)`
- Purpose: Register a compatible scenario version
- Inputs: Version string (truncated to 24 chars)
- Outputs/Return: None
- Side effects: Appends to `m_compatibleVersions` vector
- Calls: `string::string()`, `vector::push_back()`
- Notes: Silently truncates input to first 23 characters

### Scenario::IsCompatible()
- Signature: `bool IsCompatible(const string Compatible)`
- Purpose: Check if a version string is compatible with current scenario
- Inputs: Version string to validate
- Outputs/Return: `true` if empty string, matches `m_id`, or in `m_compatibleVersions`; otherwise `false`
- Side effects: None
- Calls: `vector::size()`, `operator==`
- Notes: Empty inputs are treated as compatible (permissive); linear search through compatibility list

### XML_ScenarioParser::HandleAttribute()
- Signature: `bool HandleAttribute(const char *Tag, const char *Value)`
- Purpose: Route XML attributes (name, version, id) to Scenario setter methods
- Inputs: Attribute name (Tag) and value
- Outputs/Return: `true` if recognized; `false` and calls `UnrecognizedTag()` otherwise
- Side effects: Updates Scenario singleton state via SetName/SetVersion/SetID
- Calls: `StringsEqual()`, `Scenario::instance()` (defined elsewhere)
- Notes: Calls `UnrecognizedTag()` for unmatched attributes; truncation happens in setters

### XML_CanJoinParser::HandleString()
- Signature: `bool HandleString(const char *String, int Length)`
- Purpose: Register compatible versions from `<can_join>` text nodes
- Inputs: Text content and length
- Outputs/Return: Always `true`
- Side effects: Calls `Scenario::instance()->AddCompatible()`
- Calls: `string::string()` (constructor with bounds), `Scenario::AddCompatible()`
- Notes: Converts raw C string to `std::string`; always reports success

### Scenario_GetParser()
- Signature: `XML_ElementParser *Scenario_GetParser()`
- Purpose: Factory function that assembles and returns the complete scenario XML parser
- Inputs: None
- Outputs/Return: Pointer to configured ScenarioParser root
- Side effects: Adds CanJoinParser as child of ScenarioParser
- Calls: `XML_ElementParser::AddChild()`
- Notes: Called once per application initialization; static instances persist across calls

## Control Flow Notes
**Initialization**: `Scenario_GetParser()` is called once at engine startup to configure the XML parser tree. `ScenarioParser` (root) delegates to `CanJoinParser` (child) for nested elements.

**Parsing**: When XML is read, `XML_ScenarioParser::HandleAttribute()` processes `<scenario>` attributes, and `XML_CanJoinParser::HandleString()` accumulates `<can_join>` version strings into the singleton.

**Runtime**: After parsing, `Scenario::IsCompatible()` is called during scenario loading to validate compatibility.

## External Dependencies
- `cseries.h` ΓÇö Cross-platform compatibility layer (defines `StringsEqual()`, `string` type)
- `XML_ElementParser` ΓÇö Base class for XML parsing (defined elsewhere)
- `std::string`, `std::vector` ΓÇö STL containers
- `Scenario.h` ΓÇö Header for Scenario class definition
