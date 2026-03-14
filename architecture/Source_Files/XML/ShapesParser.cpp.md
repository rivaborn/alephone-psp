# Source_Files/XML/ShapesParser.cpp

## File Purpose
Parses XML `<shape>` elements for the Aleph One game engine, extracting shape descriptor properties (collection, color lookup table, sequence/frame indices) and composing them into a single `shape_descriptor` value.

## Core Responsibilities
- Implement an XML element parser for shape definitions
- Validate and parse shape attributes (`coll`, `clut`, `seq`, `frame`)
- Enforce required attributes and acceptable bounds
- Compose validated attributes into a final `shape_descriptor` via builder macros
- Provide singleton parser instance and pointer-setter API for reuse across multiple XML elements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ShapesParser` | class | Extends `XML_ElementParser` to handle shape XML element parsing |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ShapesParser` | `XML_ShapesParser` | static | Singleton parser instance reused across multiple shape elements |

## Key Functions / Methods

### XML_ShapesParser::Start
- **Signature:** `bool Start()`
- **Purpose:** Reset parsing state at the beginning of a new shape element.
- **Inputs:** None.
- **Outputs/Return:** `true` (always succeeds).
- **Side effects:** Clears `CollPresent`, `SeqPresent` flags; resets `CLUT` to 0.
- **Calls:** None.
- **Notes:** Called once per shape element before any attributes are processed.

### XML_ShapesParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse and validate individual XML shape attributes.
- **Inputs:** Attribute tag name and string value.
- **Outputs/Return:** `true` if attribute is recognized and valid; `false` otherwise.
- **Side effects:** Updates `Coll`, `CLUT`, `Seq` members and sets `CollPresent`/`SeqPresent` flags if parsing succeeds.
- **Calls:** `StringsEqual()`, `ReadBoundedUInt16Value()`, `UnrecognizedTag()`.
- **Notes:** Supports four attributesΓÇö`coll` (collection, required), `clut` (color lookup table, optional, defaults to 0), `seq`/`frame` (sequence, required). Both `seq` and `frame` map to the same `Seq` member.

### XML_ShapesParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Validate the combination of parsed attributes and build the final shape descriptor.
- **Inputs:** None (operates on member state).
- **Outputs/Return:** `true` if validation succeeds; `false` if required attributes are missing.
- **Side effects:** Writes to `*DescPtr` (asserted non-null) via `BUILD_DESCRIPTOR()` macro.
- **Calls:** `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `AttribsMissing()` (error reporting).
- **Notes:** Requires both `coll` and `seq`/`frame` to be present, OR neither (if `NONE_Is_OK` is true, writes `UNONE`). Misses any attribute combination reports error via `AttribsMissing()`.

### Shape_GetParser
- **Signature:** `XML_ElementParser *Shape_GetParser()`
- **Purpose:** Retrieve the singleton shapes parser for use in XML parsing.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static `ShapesParser` instance.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Safe to call multiple times; always returns the same instance.

### Shape_SetPointer
- **Signature:** `void Shape_SetPointer(shape_descriptor *DescPtr, bool NONE_Is_OK = true)`
- **Purpose:** Configure the parser to write the parsed shape descriptor to a specific memory location.
- **Inputs:** Target pointer and flag indicating whether `UNONE` is a valid outcome.
- **Outputs/Return:** None.
- **Side effects:** Updates `ShapesParser.DescPtr` and `ShapesParser.NONE_Is_OK`.
- **Calls:** None.
- **Notes:** Must be called before parsing begins; `DescPtr` is asserted in `AttributesDone()`.

## Control Flow Notes
This file implements part of an XML parsing framework. Typical usage flow:
1. **Setup:** Caller invokes `Shape_SetPointer(target_addr, allow_none_flag)`.
2. **Get parser:** Caller retrieves `Shape_GetParser()` and registers it with an XML parser.
3. **Parse:** XML parser invokes `Start()`, then `HandleAttribute()` for each attribute, then `AttributesDone()`.
4. **Result:** Final `shape_descriptor` written to the target pointer (or error reported if validation fails).

## External Dependencies
- **`XML_ElementParser`** (defined elsewhere): Base class for element-specific XML parsers.
- **`shape_descriptor`** (defined elsewhere): Game engine type representing a shape reference.
- **Parsing utilities** (defined elsewhere): `ReadBoundedUInt16Value()`, `StringsEqual()`, `UnrecognizedTag()`, `AttribsMissing()`, macros `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()`, `UNONE`.
- **Constants** (defined elsewhere): `MAXIMUM_COLLECTIONS`, `MAXIMUM_CLUTS_PER_COLLECTION`, `MAXIMUM_SHAPES_PER_COLLECTION`.
