# Source_Files/XML/XML_ParseTreeRoot.h

## File Purpose
Header file declaring the root of the XML parser tree hierarchy. Provides the absolute root element that anchors all valid XML document parsers and exposes initialization/reset routines for the entire parsing structure.

## Core Responsibilities
- Export the global `RootParser` object (root of the parse tree)
- Declare `SetupParseTree()` to initialize and configure the complete parser hierarchy
- Declare `ResetAllMMLValues()` to reset all parsed values back to hard-coded defaults
- Serve as the single entry point for XML parsing initialization

## Key Types / Data Structures
None. (Uses `XML_ElementParser` from included header, but does not define new types here.)

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `RootParser` | `XML_ElementParser` | extern global | The absolute root of the entire XML parse tree; anchors all child element parsers |

## Key Functions / Methods

### SetupParseTree
- **Signature:** `extern void SetupParseTree();`
- **Purpose:** Initialize and configure the entire XML parser tree hierarchy, populating it with all valid child element parsers.
- **Inputs:** None.
- **Outputs/Return:** None (void); state change via side effect.
- **Side effects:** Populates `RootParser` with child parsers; initializes the entire parse tree structure.
- **Calls:** Not inferable from this file (implementation in corresponding .cpp file).
- **Notes:** Must be called before any XML parsing operations. Inverse of `ResetAllMMLValues()`.

### ResetAllMMLValues
- **Signature:** `extern void ResetAllMMLValues();`
- **Purpose:** Reset all values modified through XML parsing back to hard-coded defaults, effectively undoing all configuration changes.
- **Inputs:** None.
- **Outputs/Return:** None (void); state change via side effect.
- **Side effects:** Resets all parser state and parsed values across the entire tree to initial defaults.
- **Calls:** Not inferable from this file.
- **Notes:** Likely calls `ResetValues()` recursively through `RootParser` and all descendants.

## Control Flow Notes
This file is **initialization-focused**. `SetupParseTree()` must be called during engine startup before any XML files are parsed. `ResetAllMMLValues()` is a teardown or reset utility, likely used to return the engine to a clean state after configuration changes or between levels.

## External Dependencies
- **`XML_ElementParser.h`** ΓÇô base class for all element parsers; provides the hierarchical parser framework
- **`cstypes.h`** (transitively via XML_ElementParser.h) ΓÇô likely defines platform-specific integer types
