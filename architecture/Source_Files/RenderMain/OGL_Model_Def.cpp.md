# Source_Files/RenderMain/OGL_Model_Def.cpp

## File Purpose

Implements OpenGL model and skin management for the Aleph One game engine. Provides model loading from multiple 3D file formats, transformation matrix calculations, texture/skin management, and XML-based configuration parsing. Maintains a hash table for fast model-sequence lookups across collections.

## Core Responsibilities

- Manage per-collection model registry via hash table + linear search fallback
- Load and parse multiple 3D model formats (Wavefront, 3DS, Dim3, QuickDraw 3D)
- Apply transformations: rotation (X/Y/Z), scaling, shifting; compute bounding boxes
- Manage model skins/textures and OpenGL texture IDs
- Parse XML configuration for models, skins, and sequence mappings
- Handle bulk model/skin load/unload operations with progress callbacks
- Provide matrix utilities for 3├ù3 transformations (rotation, scaling, multiplication)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `SequenceMapEntry` | struct | Maps Marathon engine sequence ID to model sequence ID |
| `ModelDataEntry` | struct | Stores OGL_ModelData + sequence map + Marathon sequence for one model entry |
| `ModelHashEntry` | struct | Hash table entry: model index + sequence-map table index |
| `OGL_SkinData` | class | Skin configuration (CLUT, opacity, blend modes, image/mask filenames); inherits OGL_TextureOptionsBase |
| `OGL_SkinManager` | struct | Owns list of skins; manages OpenGL texture IDs and in-use flags |
| `OGL_ModelData` | class | Inherits OGL_SkinManager; contains model geometry, transformation params, file paths; owns Model3D instance |
| `XML_SkinDataParser` | class | Parses `<skin>` elements; updates/adds OGL_SkinData |
| `XML_SequenceMapParser` | class | Parses `<seq_map>` elements |
| `XML_MdlClearParser` | class | Parses `<model_clear>` to reset collections |
| `XML_ModelDataParser` | class | Parses `<model>` elements; aggregates skin + seq_map children |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DefaultModelData` | OGL_ModelData | static | Template for new model entries |
| `DefaultSkinData` | OGL_SkinData | static | Template for new skin entries |
| `MdlList[NUMBER_OF_COLLECTIONS]` | vector<ModelDataEntry> | static | Per-collection linear model registry |
| `MdlHash[NUMBER_OF_COLLECTIONS]` | vector<ModelHashEntry> | static | Per-collection hash table (256 slots) for fast lookup |
| `SkinDataParser` | XML_SkinDataParser | static | Reused XML parser instance for skins |
| `SequenceMapParser` | XML_SequenceMapParser | static | Reused XML parser instance for seq maps |
| `Mdl_ClearParser` | XML_MdlClearParser | static | Reused XML parser instance for model_clear |
| `ModelDataParser` | XML_ModelDataParser | static | Reused XML parser instance for models |

## Key Functions / Methods

### OGL_GetModelData
- **Signature:** `OGL_ModelData *OGL_GetModelData(short Collection, short Sequence, short& ModelSequence)`
- **Purpose:** Retrieve model data for a given collection/sequence pair; populate output parameter with resolved model sequence.
- **Inputs:** Collection ID, Marathon engine sequence ID; output reference for model sequence
- **Outputs/Return:** Pointer to OGL_ModelData if found, NULL otherwise; fills ModelSequence out-param
- **Side effects:** Lazy-initializes hash table on first call per collection; updates hash entry on cache miss
- **Calls:** Hash table lookup, linear search fallback through MdlList
- **Notes:** Hash table is built on first miss; subsequent hits avoid linear search. Two-level lookup: sequence-map table first, then neutral sequence.

### OGL_ModelData::Load
- **Signature:** `void OGL_ModelData::Load()`
- **Purpose:** Load 3D model from file, apply transformations (rotation, scaling, shift), compute transformed bounding box.
- **Inputs:** None (uses member fields: ModelFile, ModelType, XRot/YRot/ZRot, Scale, XShift/YShift/ZShift, NormalType, NormalSplit)
- **Outputs/Return:** None (modifies Model member)
- **Side effects:** Loads Model3D from disk; clears old geometry; calls OGL_SkinManager::Load()
- **Calls:** LoadModel_Wavefront, LoadModel_Studio, LoadModel_Dim3 (with two-pass), LoadModel_QD3D; matrix helpers (MatIdentity, MatMult, MatScalMult, MatVecMult); Model3D::FindPositions_Neutral, FindBoundingBox, AdjustNormals
- **Notes:** Dim3 format uses 2ΓÇô3 file passes (first for geometry, rest for frames/sequences); tries catch-blocks for optional passes. Handles both animated (FindPositions_Neutral succeeds) and static models differently: animated use transform matrices, static transform in-place. Negative scale produces mirroring.

### OGL_ModelData::Unload
- **Signature:** `void OGL_ModelData::Unload()`
- **Purpose:** Clear model geometry and unload skins.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears Model3D; calls OGL_SkinManager::Unload()
- **Calls:** Model::Clear, OGL_SkinManager::Unload
- **Notes:** Simple cleanup.

### OGL_SkinManager::Load / Unload
- **Signature:** `void Load()`, `void Unload()`
- **Purpose:** Batch load/unload all skins in SkinData vector.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Iterates skins, calls OGL_SkinData::Load/Unload on each
- **Calls:** OGL_SkinData::Load, OGL_SkinData::Unload (defined elsewhere)
- **Notes:** No-op if SkinData is empty.

### OGL_SkinManager::Reset
- **Signature:** `void Reset(bool Clear_OGL_Txtrs)`
- **Purpose:** Clear OpenGL texture IDs and in-use flags; optionally delete OpenGL texture resources.
- **Inputs:** Clear_OGL_Txtrs: whether to call glDeleteTextures
- **Outputs/Return:** None
- **Side effects:** If Clear_OGL_Txtrs, calls glDeleteTextures for all texture IDs in use. Clears IDsInUse array.
- **Calls:** glDeleteTextures (if flag set); objlist_clear
- **Notes:** Nested loop over [NUMBER_OF_OPENGL_BITMAP_SETS][NUMBER_OF_TEXTURES].

### OGL_SkinManager::GetSkin
- **Signature:** `OGL_SkinData *GetSkin(short CLUT)`
- **Purpose:** Look up skin by color lookup table ID.
- **Inputs:** CLUT ID
- **Outputs/Return:** Pointer to matching OGL_SkinData, or NULL
- **Side effects:** None
- **Calls:** None
- **Notes:** Returns first skin matching CLUT or with CLUT==ALL_CLUTS.

### OGL_SkinManager::Use
- **Signature:** `bool Use(short CLUT, short Which)`
- **Purpose:** Acquire/bind OpenGL texture; signal whether texture needs loading.
- **Inputs:** CLUT, texture index (Normal or Glowing)
- **Outputs/Return:** true if texture was just created and needs load; false if already in use
- **Side effects:** Calls glGenTextures if first use; calls glBindTexture; sets IDsInUse
- **Calls:** glGenTextures, glBindTexture
- **Notes:** Texture IDs are stored in IDs[CLUT][Which].

### OGL_LoadModels / OGL_UnloadModels
- **Signature:** `void OGL_LoadModels(short Collection)`, `void OGL_UnloadModels(short Collection)`
- **Purpose:** Bulk load/unload all models in a collection; issue progress callbacks.
- **Inputs:** Collection ID
- **Outputs/Return:** None
- **Side effects:** Iterates MdlList[Collection], calls Load/Unload on each model; calls OGL_ProgressCallback
- **Calls:** OGL_ModelData::Load/Unload, OGL_ProgressCallback
- **Notes:** LoadModels increments progress on each model; UnloadModels does not.

### OGL_ResetModelSkins
- **Signature:** `void OGL_ResetModelSkins(bool Clear_OGL_Txtrs)`
- **Purpose:** Reset skins for all models across all collections.
- **Inputs:** Clear_OGL_Txtrs: whether to delete OpenGL texture resources
- **Outputs/Return:** None
- **Side effects:** Calls OGL_SkinManager::Reset on each model in each collection
- **Calls:** OGL_SkinManager::Reset
- **Notes:** Nested loops over MAXIMUM_COLLECTIONS and MdlList.

### XML Parser Classes (XML_SkinDataParser, XML_SequenceMapParser, XML_MdlClearParser, XML_ModelDataParser)
- **Purpose:** Parse XML configuration elements (`<skin>`, `<seq_map>`, `<model_clear>`, `<model>`)
- **Inputs:** Tag/Value pairs via HandleAttribute; aggregated in AttributesDone or End
- **Outputs/Return:** true/false for success; update external containers (SkinData, SeqMap, MdlList)
- **Side effects:** Modify vectors (SkinData, SequenceMap, MdlList); call MdlDelete
- **Calls:** ReadBoundedInt16Value, ReadFloatValue, ReadInt16Value, StringsEqual, AttribsMissing, UnrecognizedTag, memcpy
- **Notes:** XML_ModelDataParser aggregates XML_SkinDataParser and XML_SequenceMapParser as children; uses AddChild/Start pattern. Parsers reuse default data templates. AttributesDone merges or appends entries.

### Matrix Helpers (MatCopy, MatIdentity, MatMult, MatScalMult, MatVecMult)
- **Purpose:** 3├ù3 matrix operations (copy, identity, multiply, scale, vector transform).
- **Inputs:** Source/dest matrices or vectors; scale factor
- **Outputs/Return:** Modify dest parameter in-place
- **Side effects:** None
- **Calls:** objlist_copy (in MatCopy)
- **Notes:** Basic linear algebra. MatMult and MatVecMult use nested loops; MatScalMult modifies in-place.

### MdlDelete / MdlDeleteAll
- **Signature:** `void MdlDelete(short Collection)`, `void MdlDeleteAll()`
- **Purpose:** Clear model registry for one or all collections.
- **Inputs:** Collection ID (for MdlDelete)
- **Outputs/Return:** None
- **Side effects:** Clears MdlList and MdlHash vectors
- **Calls:** vector::clear
- **Notes:** MdlDeleteAll loops over NUMBER_OF_COLLECTIONS.

## Control Flow Notes

This file participates in **configuration/initialization** (XML parsing) and **asset lifecycle** (load/unload):

1. **Startup/XML Parse Phase:** XML parsers parse configuration and populate MdlList and MdlHash tables.
2. **Load Phase:** OGL_LoadModels called to load models from disk; each model applies transformations and loads skins.
3. **Runtime Phase:** OGL_GetModelData called on demand to retrieve model for a sequence; hash table provides O(1) typical lookup.
4. **Unload Phase:** OGL_UnloadModels clears geometry; OGL_ResetModelSkins optionally deletes OpenGL texture resources.

No per-frame update loop; all changes driven by external calls.

## External Dependencies

- **Includes:**  
  `cseries.h` (common utility/types), `OGL_Model_Def.h` (header), `OGL_Setup.h` (config access), `cmath` (trig), `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `QD3D_Loader.h` (model loaders)

- **Defined elsewhere (used here):**  
  `OGL_TextureOptionsBase` (base class for OGL_SkinData), `Model3D` (3D geometry container), `XML_ElementParser` (XML parsing base), `FileSpecifier` (file I/O), `OGL_SkinData::Load/Unload`, `OGL_SkinData::GetMaxSize`, `Get_OGL_ConfigureData()`, `OGL_ProgressCallback()`, `objlist_copy`, `objlist_clear`, `objlist_set`, `StringsEqual`, `ReadBoundedInt16Value`, `ReadFloatValue`, `ReadInt16Value`, OpenGL functions (`glGenTextures`, `glBindTexture`, `glDeleteTextures`)

- **Constants/Enums (defined elsewhere):**  
  `NUMBER_OF_COLLECTIONS`, `MAXIMUM_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `NUMBER_OF_OPENGL_BITMAP_SETS`, `OGL_NUMBER_OF_OPACITY_TYPES`, `OGL_NUMBER_OF_BLEND_TYPES`, `Model3D::NUMBER_OF_NORMAL_TYPES`, `NUMBER_OF_MODEL_LIGHT_TYPES`, `ALL_CLUTS`, `SILHOUETTE_BITMAP_SET`, `NONE`
