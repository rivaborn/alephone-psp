# Source_Files/RenderMain/OGL_Subst_Texture_Def.cpp

## File Purpose
Implements OpenGL substitute texture definition management for walls and sprites in the Aleph One game engine. Provides hash-accelerated lookup of texture configuration (opacity, blending, scaling, positioning) and handles batch loading/unloading of texture data per collection. Also parses XML texture configuration and supports clearing/resetting definitions.

## Core Responsibilities
- Manage texture options organized by collection ID with CLUT (color lookup table) and bitmap indices
- Provide hash-table accelerated lookup of texture options via `OGL_GetTextureOptions()`
- Load and unload texture images for a collection with progress callbacks
- Parse XML `<texture>` and `<txtr_clear>` elements to configure substitute textures
- Support clearing individual collections or all texture definitions
- Cache hash lookups with separate flags for ALL_CLUTS vs. specific CLUT matches

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `OGL_TextureOptions` | struct | Inherits `OGL_TextureOptionsBase`; extends with opacity type/scale/shift, void visibility, normal/glow image paths and blending, sprite positioning (Left/Top/Right/Bottom), image scaling. |
| `TextureOptionsEntry` | struct | Wraps an `OGL_TextureOptions` with CLUT and bitmap identifiers for storage. |
| `TOList_t` | typedef | `deque<TextureOptionsEntry>` ΓÇö ordered list of texture options per collection. |
| `XML_TO_ClearParser` | class | XML element parser for `<txtr_clear>` tags; calls `TODelete()` to reset options. |
| `XML_TextureOptionsParser` | class | XML element parser for `<texture>` tags; populates and adds/replaces texture options in the list. |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultTextureOptions` | `OGL_TextureOptions` | static | Fallback default returned by `OGL_GetTextureOptions()` when no match found. |
| `TOList[NUMBER_OF_COLLECTIONS]` | `TOList_t[]` | static | Per-collection lists of texture option entries; index by collection ID. |
| `TOHash[NUMBER_OF_COLLECTIONS]` | `vector<uint16>[]` | static | Per-collection hash tables for O(1) lookup; entries store index into `TOList` plus a `Specific_CLUT_Flag` bit. |
| `TO_ClearParser` | `XML_TO_ClearParser` | static | XML parser instance for clear commands. |
| `TextureOptionsParser` | `XML_TextureOptionsParser` | static | XML parser instance for texture definitions. |

## Key Functions / Methods

### OGL_GetTextureOptions
- Signature: `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap)`
- Purpose: Retrieve texture configuration for a specific (Collection, CLUT, Bitmap) tuple. Hash-accelerated with linear search fallback.
- Inputs: Collection ID, CLUT index, Bitmap index
- Outputs/Return: Pointer to `OGL_TextureOptions` (either from list or `DefaultTextureOptions` fallback)
- Side effects: Lazily initializes hash table for collection on first lookup
- Calls: Hash function `TOHashFunc()`
- Notes: Sets/clears `Specific_CLUT_Flag` in hash entry based on whether an ALL_CLUTS entry was matched. Specific CLUT entries are prepended to list for priority matching.

### OGL_LoadTextures
- Signature: `void OGL_LoadTextures(short Collection)`
- Purpose: Load all texture images for a collection; compute sprite positioning.
- Inputs: Collection ID
- Outputs/Return: None
- Side effects: Calls `Load()` and `FindImagePosition()` on each `OGL_TextureOptions`; invokes `OGL_ProgressCallback()` per texture
- Calls: `OGL_TextureOptions::Load()`, `OGL_TextureOptions::FindImagePosition()`, `OGL_ProgressCallback()`
- Notes: Typically called during map load; progress callback used for UI feedback

### OGL_UnloadTextures
- Signature: `void OGL_UnloadTextures(short Collection)`
- Purpose: Release texture image resources for a collection.
- Inputs: Collection ID
- Outputs/Return: None
- Side effects: Calls `Unload()` on each texture option
- Calls: `OGL_TextureOptions::Unload()`

### OGL_CountTextures
- Signature: `int OGL_CountTextures(short Collection)`
- Purpose: Return the number of texture option entries in a collection.
- Inputs: Collection ID
- Outputs/Return: Size of collection's texture list
- Side effects: None

### TODelete
- Signature: `void TODelete(short Collection)`
- Purpose: Clear all texture options for a collection and reset its hash table.
- Inputs: Collection ID
- Outputs/Return: None
- Side effects: Empties `TOList[Collection]` and `TOHash[Collection]`

### XML_TextureOptionsParser::HandleAttribute / AttributesDone
- Purpose: Parse `<texture>` XML attributes (coll, clut, bitmap, opacity type/scale/shift, image paths, blending, positioning, premultiply flags)
- Inputs: Tag name and value strings
- Outputs/Return: Boolean success; calls `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadString()` to populate `OGL_TextureOptions Data`
- Side effects: On `AttributesDone()`, adds new entry to `TOList` or replaces existing match; specific CLUT entries are prepended for priority
- Notes: Requires both Collection and Bitmap attributes; CLUT defaults to ALL_CLUTS

## Control Flow Notes
**Initialization:** Hash tables are lazily initialized on first `OGL_GetTextureOptions()` call; default texture options created statically.

**Map/Level Load:** XML parser reads texture definitions, populates `TOList` and invalidates hash tables. Then `OGL_LoadTextures()` called to load images.

**Runtime Lookup:** `OGL_GetTextureOptions()` uses hash table with specific-CLUT priority over ALL_CLUTS entries.

**Shutdown/Reset:** `TODelete_All()` called (via XML `<txtr_clear>` with no coll attribute) or `OGL_UnloadTextures()` to release resources.

## External Dependencies
- `OGL_TextureOptionsBase` ΓÇö base class with opacity, blending, image/mask paths, premultiply flags (defined in `OGL_Texture_Def.h`)
- `XML_ElementParser` ΓÇö base class for XML parsing framework
- STL: `<deque>`, `vector` for dynamic collections
- Undefined here: `OGL_ProgressCallback()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadString()`, `StringsEqual()`, `ALL_CLUTS`, `NONE`, `SILHOUETTE_BITMAP_SET`, `OGL_NUMBER_OF_OPACITY_TYPES`, `OGL_NUMBER_OF_BLEND_TYPES`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `UNONE`, `TEST_FLAG()`, `SET_FLAG()`
