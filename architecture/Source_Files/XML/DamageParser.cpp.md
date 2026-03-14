# Source_Files/XML/DamageParser.cpp

## File Purpose
Implements an XML parser for damage element definitions in the Aleph One game engine. Parses damage attributes (type, flags, base, random, scale) from XML and populates a damage_definition structure. Provides a singleton parser interface for reuse across multiple damage definitions.

## Core Responsibilities
- Parse XML damage element attributes and validate their values
- Convert floating-point scale values to fixed-point representation
- Enforce bounds on damage type (0 to NUMBER_OF_DAMAGE_TYPES-1) and flags (0ΓÇô1)
- Provide a getter for the damage parser instance
- Set the target damage_definition pointer before parsing begins

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| XML_DamageParser | class | XML element parser for damage definitions; extends XML_ElementParser |
| damage_definition | struct (defined elsewhere) | Target structure holding parsed damage data (type, flags, base, random, scale) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| DamageParser | XML_DamageParser | static | Singleton parser instance reused for all damage definitions |

## Key Functions / Methods

### XML_DamageParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse individual XML damage attributes and populate the target damage_definition.
- **Inputs:** Tag name ("type", "flags", "base", "random", "scale") and string value.
- **Outputs/Return:** `true` if parse succeeds, `false` otherwise; sets fields in DamagePtr.
- **Side effects:** Modifies DamagePtr->type, flags, base, random, or scale; calls UnrecognizedTag() on unknown attributes.
- **Calls:** StringsEqual(), ReadBoundedInt16Value(), ReadInt16Value(), ReadBoundedNumericalValue(), UnrecognizedTag().
- **Notes:** Scale is converted from float to fixed-point (multiplied by FIXED_ONE). Type and flags have explicit bounds; other values are unbounded. Unknown tags trigger UnrecognizedTag() and return false.

### Damage_GetParser
- **Signature:** `XML_ElementParser *Damage_GetParser()`
- **Purpose:** Return the singleton damage parser for XML parsing.
- **Inputs:** None.
- **Outputs/Return:** Pointer to the static DamageParser instance.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Allows multiple damage elements to reuse the same parser by calling Damage_SetPointer before each parse.

### Damage_SetPointer
- **Signature:** `void Damage_SetPointer(damage_definition *DamagePtr)`
- **Purpose:** Set the target damage_definition structure for the next parse operation.
- **Inputs:** Pointer to a damage_definition struct.
- **Outputs/Return:** None (void).
- **Side effects:** Updates DamageParser.DamagePtr; all subsequent HandleAttribute calls write to this structure.
- **Calls:** None.
- **Notes:** Must be called before XML parsing begins; used to redirect parser output to different damage structures.

## Control Flow Notes
This is initialization/configuration code. Typical usage: caller invokes `Damage_SetPointer(ptr)` to designate a target structure, then the XML parser calls `Damage_GetParser()` and drives `HandleAttribute` for each attribute found in the XML. Used during game/map resource loading to populate damage definitions from XML data.

## External Dependencies
- **Includes:** `cseries.h` (standard utilities), `DamageParser.h`, `<string.h>`, `<limits.h>`
- **Defined elsewhere:** 
  - `XML_ElementParser` base class
  - `damage_definition` struct (from map.h)
  - `NUMBER_OF_DAMAGE_TYPES`, `FIXED_ONE` macros
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadBoundedNumericalValue()`, `UnrecognizedTag()` functions
  - `SHRT_MIN`, `SHRT_MAX` from `<limits.h>`
