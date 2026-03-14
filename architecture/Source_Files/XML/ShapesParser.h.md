# Source_Files/XML/ShapesParser.h

## File Purpose
This header provides a factory interface for parsing shape descriptor elements from XML configuration files. It decouples XML parsing of shape data from the shape descriptor system by offering a getter for configured parsers and a setter for the destination pointer where parsed values should be stored.

## Core Responsibilities
- Provide access to an `XML_ElementParser` instance specialized for shape element parsing
- Manage destination pointers for parsed shape descriptor values
- Support optional "NONE" values in parsed shape fields
- Abstract the shape parsing logic from callers (implementation in corresponding .cpp file)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef (uint16) | Packed descriptor combining collection, shape index, and color lookup table ID (imported from shape_descriptors.h) |

## Global / File-Static State
None.

## Key Functions / Methods

### Shape_GetParser
- **Signature:** `XML_ElementParser *Shape_GetParser()`
- **Purpose:** Obtain a parser instance for shape elements; callable multiple times to support multiple shape fields in different XML elements
- **Inputs:** None
- **Outputs/Return:** Pointer to an `XML_ElementParser` subclass configured to parse shape elements
- **Side effects:** None visible in header; likely creates/returns a singleton or cached parser from .cpp
- **Calls:** Not inferable from header
- **Notes:** Comment indicates this is reusable across multiple elements; implementation must handle repeated calls gracefully

### Shape_SetPointer
- **Signature:** `void Shape_SetPointer(shape_descriptor *DescPtr, bool NONE_Is_OK = true)`
- **Purpose:** Configure the destination pointer where the shape parser should write its parsed result
- **Inputs:** 
  - `DescPtr`: pointer to a `shape_descriptor` to receive the parsed value
  - `NONE_Is_OK`: boolean flag allowing "NONE" as a valid XML value (default: true)
- **Outputs/Return:** void; side effect is parser configuration
- **Side effects:** Modifies internal state of the shape parser to direct output to the supplied pointer; affects subsequent parse operations
- **Calls:** Not inferable from header
- **Notes:** Must be called before parsing to establish where results are written; the `NONE_Is_OK` flag likely converts "NONE" strings to a sentinel `shape_descriptor` value (e.g., all bits 1)

## Control Flow Notes
This module is part of XML configuration initialization. It does not participate directly in frame/update/render loops. Instead, it is invoked during config file loading to set up parsers for shape fields found in XML elements (likely in weapon definitions, scenery, or sprite configuration).

## External Dependencies
- **Includes:**
  - `shape_descriptors.h` ΓÇö defines `shape_descriptor` type and collection/shape macros
  - `XML_ElementParser.h` ΓÇö base class for all XML element parsers

- **External symbols used:**
  - `shape_descriptor` (typedef, defined elsewhere)
  - `XML_ElementParser` (class, defined elsewhere)
