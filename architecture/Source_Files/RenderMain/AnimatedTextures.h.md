# Source_Files/RenderMain/AnimatedTextures.h

## File Purpose
Provides the interface for animated texture management in the Aleph One rendering pipeline. Handles per-frame animation updates, texture descriptor translation (mapping static textures to animated variants), and XML-based configuration loading.

## Core Responsibilities
- Update animated texture states each frame
- Translate shape descriptors to animated equivalents during rendering
- Provide XML parser for loading animated texture configurations
- Manage the mapping between static and animated texture IDs

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `shape_descriptor` | typedef | 16-bit descriptor encoding collection, shape, and CLUT indices |

## Global / File-Static State
None.

## Key Functions / Methods

### AnimTxtr_Update
- **Signature:** `void AnimTxtr_Update()`
- **Purpose:** Advances animation state for all registered animated textures (presumably called once per frame)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies internal animation frame counters/state (not visible from header)
- **Calls:** Not inferable from this file
- **Notes:** Likely called from the main render/update loop

### AnimTxtr_Translate
- **Signature:** `shape_descriptor AnimTxtr_Translate(shape_descriptor Texture)`
- **Purpose:** Translates a texture descriptor, returning either the animated variant (if configured) or the original descriptor
- **Inputs:** `Texture` ΓÇö a shape descriptor to look up
- **Outputs/Return:** shape_descriptor ΓÇö either animated or passthrough
- **Side effects:** None apparent
- **Calls:** Not inferable from this file
- **Notes:** Called during texture binding to swap in animated variants transparently

### AnimatedTextures_GetParser
- **Signature:** `XML_ElementParser *AnimatedTextures_GetParser()`
- **Purpose:** Returns the XML element parser for animated texture configuration
- **Inputs:** None
- **Outputs/Return:** Pointer to XML_ElementParser subclass (likely for parsing `<animated_texture>` elements)
- **Side effects:** None apparent
- **Calls:** Not inferable from this file
- **Notes:** Used during map/scenario loading to configure animated textures

## Control Flow Notes
`AnimTxtr_Update()` is invoked from the main frame/update loop. `AnimTxtr_Translate()` is called during texture binding in the render path, mapping static descriptors to their animated counterparts. Configuration occurs at load time via the XML parser.

## External Dependencies
- **shape_descriptors.h** ΓÇö `shape_descriptor` typedef; defines descriptor bit layout and collection enums
- **XML_ElementParser.h** ΓÇö `XML_ElementParser` base class for parsing configuration
