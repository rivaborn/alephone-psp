# Source_Files/XML/ColorParser.h

## File Purpose
Provides an interface for parsing color elements from XML configuration files. This header defines two public functions that work together to configure color parsing: obtaining a parser instance and setting the target array where parsed color values are stored.

## Core Responsibilities
- Provides factory function to obtain an `XML_ElementParser` specialized for color elements
- Manages configuration of the target color array for parsing operations
- Supports both indexed (array-based) and non-indexed color value modes
- Handles initialization of the color parsing subsystem

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `rgb_color` | typedef/struct (defined elsewhere) | Color representation with red, green, blue components |
| `XML_ElementParser` | class (defined elsewhere) | Base class for XML element parsers |

## Global / File-Static State
None.

## Key Functions / Methods

### Color_GetParser
- Signature: `XML_ElementParser *Color_GetParser()`
- Purpose: Obtain a reusable parser instance for color XML elements
- Inputs: None
- Outputs/Return: Pointer to an `XML_ElementParser` configured to parse color elements
- Side effects: None visible; parser instance likely created or returned from a managed pool
- Calls: Not visible in header
- Notes: Documented as callable multiple times; several elements may have colors

### Color_SetArray
- Signature: `void Color_SetArray(rgb_color *ColorList, int NumColors = 0)`
- Purpose: Configure the target array where parsed color values will be stored
- Inputs: 
  - `ColorList`: pointer to an array of `rgb_color` structures
  - `NumColors`: count of colors in the array (default 0)
- Outputs/Return: None
- Side effects: Sets global/static parser state to direct parsed colors into `ColorList`
- Calls: Not visible in header
- Notes: If `NumColors <= 0`, color values are treated as non-indexed and no "index" attribute is expected in XML

## Control Flow Notes
This module is part of the XML configuration loading pipeline. Typical usage: call `Color_SetArray()` to configure the target, then use the parser from `Color_GetParser()` to parse `<color>` XML elements during initialization/configuration load.

## External Dependencies
- `#include "cseries.h"` ΓÇö Core series library (defines/declares `rgb_color` and SDL integration)
- `#include "XML_ElementParser.h"` ΓÇö XML parsing framework base class
- Defined elsewhere: `rgb_color` type, `XML_ElementParser` class implementation
