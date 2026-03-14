# Source_Files/XML/ColorParser.cpp

## File Purpose
Parses XML `<color>` elements and populates an external `rgb_color` array with parsed values.
Handles conversion of float color channels [0ΓÇô1] to 16-bit integer format, and optional indexed storage.
Provides a reusable singleton parser instance for XML pipeline integration.

## Core Responsibilities
- Define `XML_ColorParser` class as an XML element parser for the "color" tag
- Parse three color attributes: `red`, `green`, `blue` (float values)
- Optionally parse `index` attribute for array storage position
- Validate all required attributes are present before commit
- Convert and clamp float channel values [0ΓÇô1] to uint16 range [0ΓÇô65535]
- Store validated color into caller-provided array at specified index

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `XML_ColorParser` | class | Stateful XML parser for color elements; inherits from `XML_ElementParser` |
| `rgb_color` | struct/type | 16-bit per-channel color (defined elsewhere) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ColorParser` | `XML_ColorParser` | static (file) | Singleton parser instance shared across multiple parse operations |

## Key Functions / Methods

### XML_ColorParser::Start
- **Signature:** `bool Start()`
- **Purpose:** Reset parser state at the beginning of parsing a `<color>` element.
- **Inputs:** None (implicit: object state)
- **Outputs/Return:** `true` (always succeeds)
- **Side effects:** Clears `IsPresent[0ΓÇô3]` flags.
- **Calls:** None
- **Notes:** Initializes tracking array for attribute presence validation.

### XML_ColorParser::HandleAttribute
- **Signature:** `bool HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse a single XML attribute (`red`, `green`, `blue`, or `index`).
- **Inputs:** `Tag` (attribute name), `Value` (attribute value string)
- **Outputs/Return:** `true` if recognized and valid; `false` otherwise
- **Side effects:** Updates `TempColor` fields and `IsPresent` flags; may update `Index`.
- **Calls:** `StringsEqual`, `ReadBoundedInt16Value`, `ReadFloatValue`, `UnrecognizedTag`
- **Notes:** Converts float values to uint16 via `PIN(65535*CVal+0.5, 0, 65535)`. Index attribute only parsed if `NumColors > 0`.

### XML_ColorParser::AttributesDone
- **Signature:** `bool AttributesDone()`
- **Purpose:** Validate all required attributes are present and commit parsed color to array.
- **Inputs:** None (implicit: object state, `ColorList`, `NumColors`)
- **Outputs/Return:** `true` if all attributes present and stored; `false` if validation fails
- **Side effects:** Writes `TempColor` to `ColorList[Index]`.
- **Calls:** `AssertionError` (assert), `AttribsMissing`
- **Notes:** If `NumColors Γëñ 0`, index is optional and defaults to 0 (non-indexed mode).

### Color_GetParser
- **Signature:** `XML_ElementParser *Color_GetParser()`
- **Purpose:** Return the singleton color parser for XML integration.
- **Inputs:** None
- **Outputs/Return:** Pointer to static `ColorParser` instance
- **Side effects:** None
- **Calls:** None
- **Notes:** Called by XML framework to obtain parser; instance is reused across multiple color definitions.

### Color_SetArray
- **Signature:** `void Color_SetArray(rgb_color *ColorList, int NumColors)`
- **Purpose:** Configure the target array and size for color parsing.
- **Inputs:** `ColorList` (pointer to array), `NumColors` (element count; Γëñ0 means non-indexed mode)
- **Outputs/Return:** None
- **Side effects:** Updates static parser fields `ColorParser.ColorList` and `ColorParser.NumColors`.
- **Calls:** None
- **Notes:** Must be called before parsing to set destination; configures whether index attribute is required.

## Control Flow Notes
**Typical XML parsing flow:**
1. Caller invokes `Color_SetArray(array_ptr, count)` to configure target
2. XML framework obtains parser via `Color_GetParser()`
3. For each `<color>` element: `Start()` ΓåÆ `HandleAttribute()` (├ùN) ΓåÆ `AttributesDone()`
4. Loop repeats for next element; static singleton reused

## External Dependencies
- **Base class:** `XML_ElementParser` (defined elsewhere; parent for parser state machine)
- **Types:** `rgb_color` (16-bit color struct)
- **Functions (defined elsewhere):** 
  - `StringsEqual(const char*, const char*)` ΓÇô case-insensitive string comparison
  - `ReadBoundedInt16Value(const char*, int16&, int16, int16)` ΓÇô parse bounded int
  - `ReadFloatValue(const char*, float&)` ΓÇô parse float value
  - `PIN(value, min, max)` ΓÇô clamp macro
- **Includes:** `<string.h>`, `ColorParser.h`, `cseries.h`, `XML_ElementParser.h`
