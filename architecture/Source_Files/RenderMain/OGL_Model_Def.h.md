# Source_Files/RenderMain/OGL_Model_Def.h

## File Purpose
Defines OpenGL-based 3D model and skin management structures for the Aleph One game engine. Handles model file configuration, texture skinning, lighting parameters, and lifecycle management with XML configuration support.

## Core Responsibilities
- Define skin data structures (`OGL_SkinData`) that extend texture options with color-table (CLUT) targeting
- Manage multiple skins per model via `OGL_SkinManager`, maintaining OpenGL texture IDs and usage flags
- Store model metadata and transformations (`OGL_ModelData`): file paths, scaling, rotation, shifting, sidedness, normals, and lighting configuration
- Provide global functions for model collection lifecycle (count, load, unload)
- Support XML-driven model definition loading and parsing
- Interface with `Model3D` for the underlying geometry data structure

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_SkinData` | struct | Model skin configuration; extends `OGL_TextureOptionsBase` with CLUT targeting |
| `OGL_SkinManager` | struct | Manages per-model skins; owns OpenGL texture IDs for normal and glowing variants |
| `OGL_ModelData` | class | Complete model definition; extends `OGL_SkinManager` with file paths, transforms, lighting/normal/depth settings |
| Lighting mode enum | enum | Four lighting calculation modes: `OGL_MLight_Fast`, `OGL_MLight_Indiv`, with/without fade variants |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_SkinManager::Reset
- **Signature:** `void Reset(bool Clear_OGL_Txtrs);`
- **Purpose:** Clear and reinitialize skins for reloading, optionally clearing OpenGL texture resources
- **Inputs:** Boolean flag to control whether OpenGL textures are freed
- **Outputs/Return:** None
- **Side effects:** Modifies `IDs` and `IDsInUse` arrays; may deallocate OpenGL resources

### OGL_SkinManager::GetSkin
- **Signature:** `OGL_SkinData *GetSkin(short CLUT);`
- **Purpose:** Retrieve skin data for a specific color table, or shared skin for all CLUTs
- **Inputs:** CLUT index (-1 = all CLUTs)
- **Outputs/Return:** Pointer to `OGL_SkinData`, or `NULL` if not found
- **Side effects:** None

### OGL_SkinManager::Use
- **Signature:** `bool Use(short CLUT, short Which);`
- **Purpose:** Activate a skin texture variant for rendering
- **Inputs:** CLUT index, texture variant index (`Normal` or `Glowing`)
- **Outputs/Return:** `true` if skin needs loading, `false` otherwise
- **Side effects:** Updates internal texture usage state

### OGL_ModelData::ModelPresent
- **Signature:** `bool ModelPresent() {return !Model.VertIndices.empty();}`
- **Purpose:** Check if model geometry has been loaded
- **Inputs:** None
- **Outputs/Return:** `true` if vertex indices exist
- **Side effects:** None

### OGL_GetModelData
- **Signature:** `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence);`
- **Purpose:** Retrieve model definition for a game collection/sequence pair
- **Inputs:** Collection ID, sequence ID within collection; outputs matched model sequence index
- **Outputs/Return:** Pointer to model data, or `NULL` if not found
- **Side effects:** None
- **Notes:** Caller must determine if sequence is associated with a 3D model

### OGL_ResetModelSkins
- **Signature:** `void OGL_ResetModelSkins(bool Clear_OGL_Txtrs);`
- **Purpose:** Reset all loaded model skins globally; delegates to `OGL_SkinManager::Reset` for each
- **Inputs:** Boolean flag to clear OpenGL textures
- **Outputs/Return:** None
- **Side effects:** Modifies all model skin state

### OGL_LoadModels, OGL_UnloadModels
- **Signature:** `void OGL_LoadModels(short Collection);` / `void OGL_UnloadModels(short Collection);`
- **Purpose:** Load or unload all 3D models in a game collection
- **Inputs:** Collection ID
- **Outputs/Return:** None
- **Side effects:** Populates or clears model and texture data from memory

### OGL_CountModels
- **Signature:** `int OGL_CountModels(short Collection);`
- **Purpose:** Query the number of 3D models associated with a collection
- **Inputs:** Collection ID
- **Outputs/Return:** Model count
- **Side effects:** None

### ModelData_GetParser, Mdl_Clear_GetParser
- **Signature:** `XML_ElementParser *ModelData_GetParser(); XML_ElementParser *Mdl_Clear_GetParser();`
- **Purpose:** Factory functions for XML parsing objects; enable data-driven model definition loading
- **Inputs:** None
- **Outputs/Return:** Pointer to parser instance
- **Side effects:** Allocates parser object (caller owns)

## Control Flow Notes
This header is part of the engine's **initialization/data-loading** phase. Models are managed at the collection level (e.g., when a level or set of assets loads). XML configuration drives model file selection, transformations (scale, rotation, shift), and lighting/normal computation strategy. The `OGL_SkinManager` inheritance chain suggests skins and model data are co-owned.

## External Dependencies
- **OGL_Texture_Def.h**: `OGL_TextureOptionsBase` (base class for texture options)
- **Model3D.h**: `Model3D` struct (geometry, bones, frames, sequences)
- **XML_ElementParser.h**: `XML_ElementParser` (XML parsing framework)
- **OpenGL headers** (platform-conditional: `<OpenGL/gl.h>`, `<GL/gl.h>`, etc.)
- **STL**: `<vector>`
- **cstypes.h** (via XML_ElementParser.h, for integer types)
