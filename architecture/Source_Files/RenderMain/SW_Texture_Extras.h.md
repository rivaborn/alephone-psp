# Source_Files/RenderMain/SW_Texture_Extras.h

## File Purpose
Defines classes for managing software (non-hardware) textures with opacity/transparency tables in the Aleph One game engine. Provides a singleton interface to store, retrieve, and configure per-shape texture metadata across all shape collections.

## Core Responsibilities
- Store texture metadata (shape descriptor, opacity type, scale/shift parameters) for individual shapes
- Maintain opacity lookup tables (8-bit alpha values) for texture transparency
- Implement a singleton manager to organize textures by collection ID
- Support per-collection load/unload lifecycle for texture resources
- Integrate with XML-based configuration parsing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SW_Texture` | class | Single texture with opacity table and parameters; storage unit |
| `SW_Texture_Extras` | class | Singleton manager organizing textures per shape collection |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SW_Texture_Extras::m_instance` | `SW_Texture_Extras*` | static | Singleton instance for global texture manager access |

## Key Functions / Methods

### SW_Texture::SW_Texture()
- Signature: `SW_Texture() : m_opac_type(0), m_shape_descriptor(0), m_opac_scale(1.0), m_opac_shift(0.0)`
- Purpose: Initialize a texture with default settings
- Inputs: None
- Outputs/Return: Initialized object
- Side effects: None
- Calls: (initializer list)
- Notes: Default scale 1.0 (no scaling), shift 0.0 (no transformation)

### SW_Texture::opac_table()
- Signature: `uint8 *opac_table()`
- Purpose: Retrieve pointer to opacity table data
- Inputs: None
- Outputs/Return: `uint8*` pointing to opacity data, or nullptr if table is empty
- Side effects: None
- Calls: None (accesses `m_opac_table.front()`)
- Notes: Caller must check validity; returns nullptr for empty table

### SW_Texture::build_opac_table()
- Signature: `void build_opac_table()`
- Purpose: Generate/rebuild opacity table from type, scale, and shift parameters
- Inputs: None (uses member variables)
- Outputs/Return: void
- Side effects: Populates `m_opac_table` vector
- Calls: (defined elsewhere)
- Notes: Implementation not in header; called after configuration parameters are set

### SW_Texture_Extras::instance()
- Signature: `static SW_Texture_Extras *instance()`
- Purpose: Lazy-initialize and return the singleton instance
- Inputs: None
- Outputs/Return: `SW_Texture_Extras*` pointer
- Side effects: Allocates singleton via `new` on first call
- Calls: None
- Notes: No visible cleanup; assumes single lifetime for application

### SW_Texture_Extras::GetTexture() / AddTexture()
- Signatures: `SW_Texture *GetTexture(shape_descriptor)`, `SW_Texture *AddTexture(shape_descriptor)`
- Purpose: Query or create a texture entry for a shape descriptor
- Inputs: `ShapeDesc` ΓÇö shape descriptor identifying the texture
- Outputs/Return: `SW_Texture*` pointer (nullptr if not found by GetTexture)
- Side effects: AddTexture modifies `texture_list`
- Calls: (defined elsewhere)
- Notes: Implementations not in header

### SW_Texture_Extras::Load() / Unload()
- Signatures: `void Load(short Collection)`, `void Unload(short Collection)`
- Purpose: Load or unload all textures associated with a shape collection
- Inputs: `Collection` ΓÇö collection ID (0 to NUMBER_OF_COLLECTIONS-1)
- Outputs/Return: void
- Side effects: Populates or clears texture data in `texture_list[Collection]`
- Calls: (defined elsewhere)
- Notes: Lifecycle hooks for collection-based resource management

### SW_Texture_Extras_GetParser()
- Signature: `XML_ElementParser *SW_Texture_Extras_GetParser()`
- Purpose: Create/provide XML parser for texture extras configuration
- Inputs: None
- Outputs/Return: `XML_ElementParser*`
- Side effects: None visible in header
- Calls: (defined elsewhere)
- Notes: Global function; entry point for XML configuration integration

## Control Flow Notes
**Load Lifecycle:**
1. Collections are loaded via `Load(collection)` (likely during level init)
2. XML configuration parsed via `SW_Texture_Extras_GetParser()`
3. Textures added/configured via `AddTexture()` and parameter setters
4. Opacity tables built via `build_opac_table()` after parameters set
5. Textures accessed at render time via `GetTexture()`
6. Collections unloaded via `Unload(collection)` when no longer needed

The singleton pattern ensures global, persistent access throughout the application lifetime.

## External Dependencies
- `cseries.h`, `cstypes.h`: Core types (`uint8`, etc.)
- `shape_descriptors.h`: `shape_descriptor` type; `NUMBER_OF_COLLECTIONS` constant
- `XML_ElementParser.h`: Base class for XML parsing
- `<vector>`: STL vector storage
