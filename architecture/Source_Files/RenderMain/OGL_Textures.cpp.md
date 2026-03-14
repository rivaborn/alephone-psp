# Source_Files/RenderMain/OGL_Textures.cpp

## File Purpose
OpenGL texture manager for Aleph One (Marathon engine). Handles allocation, loading, rendering, and lifecycle management of textures for walls, landscapes, sprites, and 3D models. Implements special effects including infravision tinting, silhouette rendering, and VRAM-aware texture purging.

## Core Responsibilities
- Allocate and deallocate OpenGL texture handles via per-collection, per-bitmap state management
- Load textures from game resources in multiple formats (RGBA8, DXTC1/3/5 compressed)
- Generate and configure mipmaps with support for GL_SGIS_generate_mipmap extension
- Convert Marathon color tables to OpenGL RGBA 8888 format
- Apply infravision tinting (grayscale intensity with per-collection color overlay)
- Render silhouettes (make all opaque with tint color)
- Adjust pixel opacity via scaling, shifting, or color-based calculation (Tomb Raider hack)
- Load substitute textures with dimension validation and scaling calculations
- Age-track textures and purge unused ones per type (walls: 10s, sprites: 15s, weapons: 20s)
- Manage texture matrices for coordinate space transformations (rotation, scaling, translation)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `TextureState` | class | Manages OpenGL texture handles and loading state for a single texture with Normal/Glowing variants |
| `CollBitmapTextureState` | struct | Per-color-table texture state within a collection-bitmap group |
| `TxtrTypeInfoData` | struct | OpenGL filtering, resolution, and color format configuration per texture type |
| `InfravisionData` | struct | RGB tint components and enable flag per collection for infravision effect |
| `TextureManager` | class | Main interface for texture setup and rendering; handles shape descriptors and transfer modes |
| `OGL_TexturesStats` | struct | Global statistics: binds, age tracking, min/max bind times |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gGLTxStats` | `OGL_TexturesStats` | global | Texture statistics (bind counts, age, timing metrics) |
| `TxtrTypeInfoList[OGL_NUMBER_OF_TEXTURE_TYPES]` | `TxtrTypeInfoData[]` | static | Per-type OpenGL filter and format configuration |
| `ModelSkinInfo` | `TxtrTypeInfoData` | static | Configuration for 3D model skin textures |
| `useSGISMipmaps` | bool | static | Whether GL_SGIS_generate_mipmap extension is available |
| `IVDataList[NUMBER_OF_COLLECTIONS]` | `InfravisionData[]` | static | Per-collection infravision tint colors |
| `InfravisionActive` | bool | static | Global infravision on/off flag |
| `sgActiveTextureStates` | `list<TextureState*>` | static | Doubly-linked list of allocated texture states for age tracking |
| `TextureStateSets[OGL_NUMBER_OF_TEXTURE_TYPES][MAXIMUM_COLLECTIONS]` | `CollBitmapTextureState**` | static | 2D array mapping texture type and collection to per-bitmap texture state arrays |

## Key Functions / Methods

### TextureState::Allocate
- Signature: `bool Allocate(short txType)`
- Purpose: Allocate OpenGL texture handle(s) if not already allocated
- Inputs: `txType` ΓÇö texture type (OGL_Txtr_Wall, OGL_Txtr_Landscape, etc.)
- Outputs/Return: `true` if new allocation; `false` if already allocated
- Side effects: Calls `glGenTextures(NUMBER_OF_TEXTURES, IDs)`, appends to `sgActiveTextureStates`, increments `gGLTxStats.inUse`
- Calls: `glGenTextures`, list::push_front
- Notes: Idempotent; first call allocates, subsequent calls return false

### TextureState::Use
- Signature: `bool Use(int Which)`
- Purpose: Bind texture and report whether it requires loading
- Inputs: `Which` ΓÇö texture index (Normal or Glowing, from enum in TextureState)
- Outputs/Return: `true` if texture not yet generated (load needed); `false` if already generated
- Side effects: Calls `glBindTexture(GL_TEXTURE_2D, IDs[Which])`, sets `TexGened[Which] = true`, increments `IDUsage[Which]`
- Calls: `glBindTexture`

### TextureState::Reset
- Signature: `void Reset()`
- Purpose: Deallocate OpenGL texture handles and reset state
- Side effects: Calls `glDeleteTextures(NUMBER_OF_TEXTURES, IDs)` if `IsUsed`, removes from `sgActiveTextureStates`, clears flags
- Calls: `glDeleteTextures`, list::remove

### TextureState::FrameTick
- Signature: `void FrameTick()`
- Purpose: Track texture age and purge from VRAM if unused beyond type-specific timeout
- Side effects: Increments `unusedFrames` if no usage this frame; calls `Reset()` when timeout exceeded
- Notes: Walls timeout at 300 frames (~10 seconds), sprites at 450 (15s), weapons at 600 (20s); landscapes never purged; invoked from `OGL_FrameTickTextures()` each frame

### OGL_StartTextures
- Signature: `void OGL_StartTextures()`
- Purpose: Initialize texture management system at engine startup
- Outputs: Initializes all static state: `TxtrTypeInfoList`, `ModelSkinInfo`, texture statistics
- Side effects: Allocates `TextureStateSets` 2D array with dynamic sizing per collection; calls `OGL_CheckExtension("GL_SGIS_generate_mipmap")`; calls `glGenTextures(1, &OGL_Term_Texture)` on SDL
- Calls: `is_collection_present`, `get_number_of_collection_bitmaps`, `Get_OGL_ConfigureData`, `OGL_CheckExtension`, `glGenTextures`

### OGL_StopTextures
- Signature: `void OGL_StopTextures()`
- Purpose: Shut down texture management and release resources
- Side effects: Calls `delete[]` on all `TextureStateSets[it][ic]`; calls `glDeleteTextures(1, &OGL_Term_Texture)` on SDL
- Calls: `glDeleteTextures`

### OGL_FrameTickTextures
- Signature: `void OGL_FrameTickTextures()`
- Purpose: Per-frame texture lifecycle management
- Side effects: Iterates `sgActiveTextureStates` and calls `FrameTick()` on each
- Calls: `TextureState::FrameTick()`

### TextureManager::Setup
- Signature: `bool Setup()`
- Purpose: Prepare a texture for rendering by loading/substituting resources and configuring parameters
- Inputs: Member variables `ShapeDesc`, `TransferMode`, `TextureType`, `LowLevelShape`, `Landscape_AspRatExp`
- Outputs/Return: `true` if setup succeeded; `false` if texture not found or invalid
- Side effects: Parses shape descriptor, loads substitute or source textures, generates color tables, calculates sprite scale/offset, may call `Minify()` to fit texture constraints, updates `TxtrStatePtr` and `TxtrOptsPtr`
- Calls: `ModifyCLUT`, `get_bitmap_index`, `LoadSubstituteTexture`, `SetupTextureGeometry`, `FindColorTables`, `IsLandscapeFlatColored`, `GetFakeLandscape`, `GetOGLTexture`, `glGetIntegerv(GL_MAX_TEXTURE_SIZE)`
- Notes: Fetches from cache or allocates new `CollBitmapTextureState` and `TextureState`; coordinate space may be transposed (sprite vs wall convention)

### TextureManager::RenderNormal / RenderGlowing
- Signature: `void RenderNormal()` / `void RenderGlowing()`
- Purpose: Render normal and glowing texture variants; must call `RenderNormal` first
- Side effects: Calls `TxtrStatePtr->Allocate()` (NormalOnly), binds texture, calls `PlaceTexture()` if loaded, updates statistics
- Calls: `TextureState::Allocate`, `TextureState::UseNormal` / `TextureState::UseGlowing`, `PlaceTexture`, (implicit `glBindTexture`)
- Notes: Timing statistics collected but not actually used (`time=0` hardcoded)

### LoadSubstituteTexture
- Signature: `bool LoadSubstituteTexture()`
- Purpose: Load custom replacement texture from external source if available
- Outputs/Return: `true` if substitute loaded; `false` if none available or invalid
- Side effects: Sets `TxtrWidth`, `TxtrHeight`, `U_Scale`, `V_Scale`, `U_Offset`, `V_Offset`; validates power-of-2 for walls and horizontal for landscapes; clears `NormalBuffer` / `GlowBuffer`; sets `TxtrOptsPtr->Substitution = true`
- Calls: `NextPowerOfTwo`, (implicit `ImageDescriptor` copy-on-write)
- Notes: Per-type validation: walls/sprites require both dimensions power-of-2; landscapes require width power-of-2 only; validates against patched bitmaps (returns false if `Texture->flags & _PATCHED_BIT`)

### FindOGLColorTable
- Signature: `static void FindOGLColorTable(int NumSrcBytes, byte *OrigColorTable, uint32 *ColorTable)`
- Purpose: Convert Marathon color tables (ARGB 5551 or ARGB 8888) to OpenGL RGBA 8888 format
- Inputs: `NumSrcBytes` (2 or 4), pointer to original table, pointer to output table
- Outputs: Writes `MAXIMUM_SHADING_TABLE_INDEXES` RGBA values to `ColorTable`
- Side effects: Converts endianness and bit layout; makes all colors opaque (alpha = 0xff)
- Notes: For 2-byte: ARGB 5551 ΓåÆ RGBA 8888; for 4-byte: ARGB 8888 ΓåÆ RGBA 8888; endianness-aware via `ALEPHONE_LITTLE_ENDIAN` macro

### FindInfravisionVersionRGBA (two overloads)
- Signature: `void FindInfravisionVersionRGBA(short Collection, GLfloat *Color)` / `void FindInfravisionVersionRGBA(short Collection, int NumPixels, uint32 *Pixels)`
- Purpose: Apply infravision effect by calculating grayscale intensity and re-tinting with collection-specific color
- Inputs: `Collection` index, color(s) as float or uint32 (RGBA)
- Outputs: Modified colors in-place
- Side effects: Replaces RGB channels with `IVData.Red/Green/Blue * (R+G+B)/3`; preserves alpha
- Notes: Inactive if `!InfravisionActive` or `!IVData.IsTinted`; float version used for single colors, uint32 version optimized for texture buffers; uses marching-pointer optimization

### LoadModelSkin
- Signature: `void LoadModelSkin(ImageDescriptor& SkinImage, short Collection, short CLUT)`
- Purpose: Load and configure 3D model skin texture with optional infravision/silhouette processing
- Inputs: `SkinImage` (image data), `Collection` index, `CLUT` (color table or INFRAVISION_BITMAP_SET, SILHOUETTE_BITMAP_SET)
- Side effects: Calls `FindInfravisionVersion` or `FindSilhouetteVersion`, may minify image, calls `glTexImage2D` or `glCompressedTexImage2DARB`, sets texture parameters (`GL_TEXTURE_WRAP_S/T` to `GL_CLAMP`)
- Calls: `glGetIntegerv(GL_MAX_TEXTURE_SIZE)`, `glTexImage2D`, `glCompressedTexImage2DARB`, `gluBuild2DMipmaps`, `glTexParameteri`, `glTexEnvi`
- Notes: Supports RGBA8, DXTC1/3/5 formats; builds mipmaps if requested or generates via SGIS extension; clamps both S and T coordinates for model skins (different from landscape tiling)

### SetPixelOpacities (dispatcher)
- Signature: `void SetPixelOpacities(OGL_TextureOptions& Options, ImageDescriptorManager &imageManager)`
- Purpose: Adjust pixel alpha channel based on opacity options (scale, shift, calculation mode)
- Inputs: `Options` (OpacityScale, OpacityShift, OpacityType), `imageManager` (image data)
- Side effects: Modifies alpha channel; may call `MakeRGBA()` to decompress DXTC if `OpacityType` is Avg/Max; delegates to `SetPixelOpacitiesRGBA`, `SetPixelOpacitiesDXTC3`, `SetPixelOpacitiesDXTC5`
- Notes: Early-returns if no-op (default scale 1.0, shift 0.0, and type is neither Avg nor Max)

### SetPixelOpacitiesRGBA
- Signature: `void SetPixelOpacitiesRGBA(OGL_TextureOptions& Options, int NumPixels, uint32 *Pixels)`
- Purpose: Adjust RGBA pixel opacity by color-based calculation or scale/shift
- Inputs: `Options`, number of pixels, RGBA pixel array (8 bits per channel)
- Outputs: Modified alpha channel only
- Side effects: Computes opacity from R/G/B average (OGL_OpacType_Avg) or max (OGL_OpacType_Max), or uses existing alpha; scales by `OpacityScale`, shifts by `OpacityShift * 255`, clamps to [0, 255]
- Notes: "Tomb Raider texture-opacity hack" ΓÇö allows using color brightness to drive opacity

## Control Flow Notes
**Init Phase:** `OGL_StartTextures()` allocates and configures texture type parameters at engine startup.

**Per-Texture Setup:** `TextureManager::Setup()` invoked when a texture is needed; loads substitute or source data, computes scaling/offset, sets up color tables.

**Per-Frame Rendering:** `RenderNormal()` allocates handles and loads texture; `RenderGlowing()` optionally renders glow; `SetupTextureMatrix()` applies coordinate transforms (90┬░ rotation + y-flip for sprites, y-scale + translate for landscapes).

**Per-Frame Maintenance:** `OGL_FrameTickTextures()` ages active textures; `TextureState::FrameTick()` purges unused textures based on type-specific timeout thresholds.

**Shutdown:** `OGL_StopTextures()` deallocates all texture state arrays and OpenGL resources.

## External Dependencies
- **OpenGL:** `glGenTextures`, `glDeleteTextures`, `glBindTexture`, `glTexImage2D`, `glCompressedTexImage2DARB`, `gluBuild2DMipmaps`, `glGetIntegerv`, `glTexParameteri`, `glTexEnvi`, `glMatrixMode`, `glLoadIdentity`, `glRotatef`, `glScalef`, `glTranslatef`
- **Marathon Engine:** `get_bitmap_index`, `get_number_of_collection_bitmaps`, `is_collection_present`, `GET_DESCRIPTOR_COLLECTION/SHAPE`, `GET_COLLECTION_CLUT`, `OGL_GetTextureOptions`, `Get_OGL_ConfigureData`, `OGL_CheckExtension`, `OGL_IsActive`, `OGL_ResetModelSkins`
- **Image/Texture:** `ImageDescriptor`, `ImageDescriptorManager`, `OGL_TextureOptions`
- **SDL:** `SDL_SwapLE16`, `SDL_Color` (for endianness, color type)
