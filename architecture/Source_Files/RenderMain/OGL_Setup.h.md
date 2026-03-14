# Source_Files/RenderMain/OGL_Setup.h

## File Purpose
OpenGL initialization and configuration header for the Aleph One game engine. Provides functions to detect OpenGL presence, manage rendering configuration (textures, fog, 3D models), and handle XML-based configuration parsing. Supports dynamic loading/unloading of assets and rendering feature flags.

## Core Responsibilities
- OpenGL presence detection and initialization on the host system
- Global rendering configuration management (texture quality, flags, colors, anisotropy)
- Texture configuration per type (walls, landscapes, inhabitants, weapons in hand)
- 3D model and skin data management and lifecycle
- Fog rendering configuration and data storage
- Progress tracking callbacks during asset loading operations
- XML configuration parsing support
- Dynamic asset (models, images, textures) loading/unloading by collection
- Texture reset/reload functionality (for texture corruption recovery)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_Texture_Configure` | struct | Holds filtering, resolution, and color format settings for a single texture category |
| `OGL_ConfigureData` | struct | Global OpenGL configuration container: per-type texture configs, model config, rendering flags, void/landscape colors, anisotropy, and GeForce-specific fixes |
| `OGL_FogData` | struct | Fog configuration: color, depth (World Units), presence flag, and landscape applicability |

**Enumerations**:
- `OGL_Txtr_*`: Texture type categories (Wall, Landscape, Inhabitant, WeaponsInHand)
- `OGL_Flag_*`: Rendering feature flags (ZBuffer, VoidColor, FlatLand, Fog, 3D_Models, 2DGraphics, FlatStatic, Fader, LiqSeeThru, Map, TextureFix, HUD)
- `OGL_Fog_*`: Fog layer types (AboveLiquid, BelowLiquid)

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `OGL_ConfigureData` (via accessor) | `OGL_ConfigureData` | global/singleton | Global rendering configuration state managed via `Get_OGL_ConfigureData()` |
| Progress state | implicit | static | Progress tracking counters/state managed by progress functions |

## Key Functions / Methods

### OGL_Initialize
- **Signature:** `bool OGL_Initialize()`
- **Purpose:** Detect OpenGL availability and perform engine-level initialization
- **Return:** Boolean indicating success

### OGL_IsPresent
- **Signature:** `bool OGL_IsPresent()`
- **Purpose:** Query whether OpenGL is available on the host system
- **Return:** Boolean

### OGL_IsActive
- **Signature:** `bool OGL_IsActive()`
- **Purpose:** Query whether OpenGL is currently active for rendering
- **Return:** Boolean

### OGL_CheckExtension
- **Signature:** `bool OGL_CheckExtension(const std::string)`
- **Purpose:** Check if a specific OpenGL extension string is supported
- **Return:** Boolean

### OGL_StartProgress / OGL_ProgressCallback / OGL_StopProgress
- **Purpose:** Progress tracking during long-running asset loading (e.g., texture/model loads)
- **Inputs:** `total_progress`, `delta_progress` (incremental)
- **Side effects:** Updates internal progress state; likely triggers UI updates

### Get_OGL_ConfigureData
- **Signature:** `OGL_ConfigureData& Get_OGL_ConfigureData()`
- **Purpose:** Accessor for the global OpenGL configuration structure
- **Return:** Reference to global config (allows modification)

### OGL_SetDefaults
- **Signature:** `void OGL_SetDefaults(OGL_ConfigureData& Data)`
- **Purpose:** Initialize configuration struct with sensible engine defaults
- **Inputs:** Reference to config struct to populate

### OGL_CountModelsImages / OGL_LoadModelsImages / OGL_UnloadModelsImages
- **Purpose:** Lifecycle management for 3D models and associated images per collection (character/object type)
- **Inputs:** `short Collection` (collection identifier)
- **Return:** `OGL_CountModelsImages` returns count
- **Side effects:** Allocation/deallocation of model and texture resources

### OGL_ResetTextures
- **Signature:** `void OGL_ResetTextures()`
- **Purpose:** Clear all texture memory and force reload on next use (workaround for texture corruption)
- **Implementation note:** Defined in OGL_Textures.cpp

### OGL_GetFogData
- **Signature:** `OGL_FogData *OGL_GetFogData(int Type)`
- **Purpose:** Retrieve fog configuration for a specific layer (above/below liquid)
- **Inputs:** Fog type enum
- **Return:** Pointer to fog data (or NULL if not found)

### OpenGL_GetParser
- **Signature:** `XML_ElementParser *OpenGL_GetParser()`
- **Purpose:** Get XML element parser for reading OpenGL configuration from files
- **Return:** Parser object for hierarchical configuration parsing

## Control Flow Notes

**Initialization Phase:**
- `OGL_Initialize()` called at engine startup
- `OGL_IsPresent()` and `OGL_IsActive()` used to gate OpenGL-specific rendering paths
- `OGL_SetDefaults()` populates initial configuration
- `OpenGL_GetParser()` used to load XML configuration overrides

**Asset Loading Phase:**
- `OGL_StartProgress()` signals start of bulk asset load
- `OGL_LoadModelsImages()` / `OGL_LoadTextures()` (from included headers) called per collection
- `OGL_ProgressCallback()` invoked repeatedly as loading progresses
- `OGL_StopProgress()` signals completion

**Gameplay/Rendering Phase:**
- `Get_OGL_ConfigureData()` read by renderer to apply configuration
- Flags in `OGL_ConfigureData.Flags` control feature availability (fog, flat landscapes, 2D graphics via OpenGL, etc.)
- `OGL_ResetTextures()` callable if texture corruption detected
- `OGL_GetFogData()` called to retrieve per-layer fog parameters

## External Dependencies

**Included headers:**
- `XML_ElementParser.h` ΓÇö XML configuration hierarchical parsing framework
- `OGL_Subst_Texture_Def.h` ΓÇö Texture option definitions and texture loading/unloading
- `OGL_Model_Def.h` ΓÇö 3D model and skin data definitions; model lifecycle management
- `<string>` ΓÇö STL string class for extension name checking

**Conditional includes:**
- OpenGL platform headers (conditional on `__WIN32__`, `__APPLE__` / `__MACH__`, SDL, etc.)

**Defined elsewhere:**
- `OGL_ResetModelSkins()` ΓÇö model skin reset (conditionally declared under `#ifdef HAVE_OPENGL`)
- Texture configuration accessors (`OGL_GetTextureOptions()` in OGL_Subst_Texture_Def.h)
- Model data accessors (`OGL_GetModelData()` in OGL_Model_Def.h)

**Notes:**
- Conditional compilation on `HAVE_OPENGL` and platform-specific macros (`__WIN32__`, `__APPLE__`, SDL)
- Define `OPENGL_DOESNT_COPY_ON_SWAP` on Win32 and macOS
- Code sections marked `MOVED_OUT` indicate historical design that has been moved/refactored
