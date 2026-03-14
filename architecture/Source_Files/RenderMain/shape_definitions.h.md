# Source_Files/RenderMain/shape_definitions.h

## File Purpose
Defines the data structure for collection headers that manage shape/sprite asset collections in the rendering system. Acts as the primary interface between the asset loader and the rendering pipeline, storing metadata and pointers to collection definitions and shading tables.

## Core Responsibilities
- Define the `collection_header` struct (32 bytes) to store collection metadata on disk
- Track offsets and lengths for both standard and 16-bit format collection data
- Maintain pointers to runtime collection definitions and shading lookup tables
- Provide the global array of collection headers indexed by collection ID

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `collection_header` | struct | Container for collection metadata: flags, offsets, data lengths, and runtime pointers to collection and shading data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `collection_headers` | `struct collection_header[MAXIMUM_COLLECTIONS]` | static | Central registry of all loaded collections; indexed by collection ID |

## Key Functions / Methods
None ΓÇö this is a data structure definition file.

## Control Flow Notes
- Part of the asset/collection loading pipeline
- The `collection_header` struct bridges disk format (offsets/lengths) with runtime pointers
- The global `collection_headers` array is likely populated at engine initialization and accessed during frame rendering to fetch shading and geometry data
- The dual offset/length fields (`offset`/`length` vs. `offset16`/`length16`) suggest support for two different shape formats or compression schemes

## External Dependencies
- **Defined elsewhere**: `collection_definition` (referenced via pointer), `byte` and integer type aliases (`int16`, `uint16`, `int32`)
- **Assumed external**: `MAXIMUM_COLLECTIONS` constant (likely defined in a config or constants header)

---

**Notes:**
- The structure is exactly 32 bytes as documented, matching a likely fixed-size disk format
- The comment from Aug 2000 indicates a refactoring from MacOS-specific handles to pointers for cross-platform compatibility
- Status and flags fields suggest collection lifecycle management (loaded, active, etc.)
