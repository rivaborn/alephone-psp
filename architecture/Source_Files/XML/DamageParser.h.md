# Source_Files/XML/DamageParser.h

## File Purpose
Interface for parsing XML "damage" elements into `damage_definition` structures. Provides factory and target-setting functions for an XML parser specialized in damage configuration.

## Core Responsibilities
- Expose factory function to obtain a damage XML parser instance
- Define mechanism to specify the target `damage_definition` object for parsing results
- Abstract XML parsing details for damage-related game data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `damage_definition` | struct (defined in map.h) | Target structure containing damage type, flags, base/random values, and scaling |

## Global / File-Static State
None.

## Key Functions / Methods

### Damage_GetParser
- **Signature:** `XML_ElementParser *Damage_GetParser();`
- **Purpose:** Factory function to obtain a new damage element parser
- **Inputs:** None
- **Outputs/Return:** Pointer to an `XML_ElementParser` configured for parsing damage elements
- **Side effects:** Allocates parser instance (caller responsible for cleanup)
- **Calls:** Not inferable from this file
- **Notes:** Returns an instance capable of parsing XML attributes: `type`, `flags`, `base`, `random`, `scale` (all optional)

### Damage_SetPointer
- **Signature:** `void Damage_SetPointer(damage_definition *DamagePtr);`
- **Purpose:** Configure the target destination for parsed damage data
- **Inputs:** `DamagePtr` ΓÇô pointer to a `damage_definition` struct to populate
- **Outputs/Return:** None
- **Side effects:** Sets internal parser state to write into the specified damage structure
- **Calls:** Not inferable from this file
- **Notes:** Must be called before parsing to direct results into the correct struct

## Control Flow Notes
Part of the resource/map initialization pipeline. Likely invoked during XML configuration loading to parse damage type definitions. Follows a two-phase pattern: (1) get parser instance, (2) set target struct, (3) parse XML (handled by caller).

## External Dependencies
- **map.h** ΓÇô `damage_definition` struct, damage type/flag enums
- **XML_ElementParser.h** ΓÇô base parser class that this module wraps
