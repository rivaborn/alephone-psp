# Source_Files/RenderMain/OGL_Textures.h

## File Purpose
OpenGL texture management header for the Aleph One game engine (Marathon port). Provides texture accounting, state management, and rendering operations for both normal and special-effect textures (glow maps, infravision, silhouette).

## Core Responsibilities
- Initialize, manage, and tear down texture accounting and resource pools
- Track per-frame texture usage and perform housekeeping (aging unused textures)
- Manage texture state for individual texture sets (normal and glow variants)
- Set up texture geometry, color tables, and coordinate transformations
- Handle texture rendering with support for opacity blending and special effects
- Implement color-table modifications for infravision and silhouette rendering modes
- Convert pixel formats (16-bit to 32-bit) and normalize colors for OpenGL
- Load and manage model skins and landscape textures

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| TextureState | struct | Per-texture-set state tracking; holds IDs, usage flags, generation status, and frame-age counter |
| CollBitmapTextureState | struct | Aggregates TextureState for a collection bitmap; includes UV scaling/offset for sprite padding |
| TextureManager | class | Central texture manager; handles setup, geometry calculation, buffer allocation, and rendering for both normal and glow textures |
| OGL_TexturesStats | struct | Statistics tracker (bind counts, setup times, texture age) for profiling and debugging |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| BadTextureType | const int | global | Sentinel value (32767) for "no texture" default argument |
| gGLTxStats | OGL_TexturesStats | extern | Runtime statistics on texture binding, setup duration, and age |
| ConversionTable_16to32 | GLuint* | extern (commented) | Cached lookup table for 16ΓåÆ32-bit pixel conversion (optional optimization) |

## Key Functions / Methods

### OGL_StartTextures()
- Signature: `void OGL_StartTextures()`
- Purpose: Initialize texture accounting and resource pools at render session start
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates texture manager state; global texture accounting reset
- Calls: Not visible in this header
- Notes: Must be called before any texture operations

### OGL_StopTextures()
- Signature: `void OGL_StopTextures()`
- Purpose: Tear down texture accounting and free resources
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates texture resources; clears global state
- Calls: Not visible in this header
- Notes: Should be called at render session end

### OGL_FrameTickTextures()
- Signature: `void OGL_FrameTickTextures()`
- Purpose: Per-frame housekeeping (e.g., aging unused textures, evicting from cache)
- Inputs: None
- Outputs/Return: None
- Side effects: Updates texture usage counters; may deallocate stale textures
- Calls: Likely calls TextureState::FrameTick() for each texture
- Notes: Called every frame

### TextureState::Allocate()
- Signature: `bool Allocate(short txType)`
- Purpose: Allocate OpenGL texture IDs for the given texture type
- Inputs: `txType` ΓÇô texture type identifier
- Outputs/Return: `bool` ΓÇô true if allocation succeeded
- Side effects: Allocates OpenGL texture IDs; sets TexGened flags
- Calls: OpenGL texture allocation functions (not visible)
- Notes: May fail if texture types or resources are exhausted

### TextureState::Use() / UseNormal() / UseGlowing()
- Signature: `bool Use(int Which)`, `bool UseNormal()`, `bool UseGlowing()`
- Purpose: Mark a texture variant (Normal or Glowing) as in-use this frame
- Inputs: `Which` ΓÇô Normal or Glowing enum value
- Outputs/Return: `bool` ΓÇô true if texture is usable
- Side effects: Updates IDUsage counter; resets unusedFrames timer
- Calls: None visible
- Notes: Glow variant only valid if IsGlowing is true

### TextureManager::Setup()
- Signature: `bool Setup()`
- Purpose: Set up all texture state from input descriptors (ShapeDesc, Texture, etc.)
- Inputs: Member variables (ShapeDesc, LowLevelShape, Texture, ShadingTables, TransferMode, etc.)
- Outputs/Return: `bool` ΓÇô true if setup succeeded
- Side effects: Populates geometry, color tables, buffers; may allocate textures
- Calls: SetupTextureGeometry(), FindColorTables(), GetOGLTexture(), LoadSubstituteTexture(), PlaceTexture(), PremultiplyColorTables()
- Notes: Must be called before rendering; complex multi-step initialization

### TextureManager::RenderNormal() / RenderGlowing()
- Signature: `void RenderNormal()`, `void RenderGlowing()`
- Purpose: Apply normal texture to OpenGL (RenderNormal safe for allocation; RenderGlowing follows it)
- Inputs: None (uses member state from Setup())
- Outputs/Return: None
- Side effects: Binds texture ID to OpenGL; updates gGLTxStats
- Calls: OpenGL texture binding and matrix operations (not fully visible)
- Notes: RenderNormal allocates; always call first. RenderGlowing only valid if IsGlowing is true

### ModifyCLUT()
- Signature: `short ModifyCLUT(short TransferMode, short CLUT)`
- Purpose: Modify color-table index for special modes (infravision, silhouette)
- Inputs: `TransferMode` ΓÇô rendering mode; `CLUT` ΓÇô original color-table index
- Outputs/Return: Modified color-table index
- Side effects: None
- Calls: None visible
- Notes: Mapping is mode-dependent

### Convert_16to32()
- Signature: `inline GLuint Convert_16to32(uint16 InPxl)`
- Purpose: Convert 16-bit 1555 ARGB pixel to 32-bit RGBA with opaque alpha
- Inputs: 16-bit pixel (ARGB 1555)
- Outputs/Return: 32-bit pixel (RGBA 8888, alpha = 0xff or 0xff000000 depending on endianness)
- Side effects: None
- Calls: FiveToEight() (inline helper)
- Notes: Endianness-aware; alpha always preset to fully opaque

### MakeFloatColor()
- Signature: `inline void MakeFloatColor(RGBColor& InColor, GLfloat *OutColor)` (overloaded for rgb_color)
- Purpose: Normalize 16-bit RGB color components to [0.0, 1.0] floating-point range
- Inputs: InColor (RGBColor or rgb_color); OutColor pointer (3-element array)
- Outputs/Return: Writes normalized values to OutColor
- Side effects: None (writes to output pointer)
- Calls: None
- Notes: Divides each 16-bit channel by 65535.0F

### LoadModelSkin()
- Signature: `void LoadModelSkin(ImageDescriptor& Image, short Collection, short CLUT)`
- Purpose: Load a model skin image from collection+CLUT
- Inputs: Image (output descriptor), Collection ID, CLUT ID
- Outputs/Return: None (Image is populated)
- Side effects: Loads image data; may read from resource files
- Calls: Not visible
- Notes: Used for model rendering

### SetPixelOpacities() / SetPixelOpacitiesRGBA()
- Signature: `void SetPixelOpacities(OGL_TextureOptions& Options, ImageDescriptorManager &imageManager)`, `void SetPixelOpacitiesRGBA(OGL_TextureOptions& Options, int NumPixels, uint32 *Pixels)`
- Purpose: Adjust alpha channel based on texture options
- Inputs: Options (opacity configuration), image data (either ImageDescriptorManager or raw RGBA pixel array)
- Outputs/Return: None (modifies in-place)
- Side effects: Modifies pixel alpha values
- Calls: Not visible
- Notes: Variant for raw RGBA arrays accepts pixel count

### FindInfravisionVersion*() / FindSilhouetteVersion()
- Signature: `void FindInfravisionVersion(short Collection, ImageDescriptorManager &imageManager)`, `void FindInfravisionVersionRGBA(short Collection, GLfloat *Color)`, `void FindInfravisionVersionRGBA(short Collection, int NumPixels, uint32 *Pixels)`, `void FindSilhouetteVersion(ImageDescriptorManager &imageManager)`
- Purpose: Apply color tinting for infravision or silhouette rendering modes
- Inputs: Collection ID, image/pixel data (overloads for different formats)
- Outputs/Return: None (modifies in-place)
- Side effects: Recolors pixels or applies tint
- Calls: Not visible
- Notes: Multiple overloads for color, pixel array, and image descriptor; IsInfravisionActive() checks if effect is enabled

### SetInfravisionTint()
- Signature: `bool SetInfravisionTint(short Collection, bool IsTinted, float Red, float Green, float Blue)`
- Purpose: Configure infravision tint color for a shapes collection
- Inputs: Collection ID, enable/disable flag, RGB tint values [0.0, 1.0]
- Outputs/Return: `bool` ΓÇô success
- Side effects: Updates tint configuration for the collection
- Calls: Not visible
- Notes: Per-collection setting; enables/disables tinting globally with IsTinted flag

## Control Flow Notes

**Initialization**: OGL_StartTextures() called once at render session start; TextureManager instances created per-texture as needed.

**Per-Frame**: OGL_FrameTickTextures() called each frame to age and evict unused textures. Individual TextureState::FrameTick() updates usage counters.

**Texture Setup**: TextureManager::Setup() called when a new texture is encountered (populates all geometry, color tables, and buffers).

**Rendering**: TextureManager::RenderNormal() binds the texture ID and applies transformations; TextureManager::RenderGlowing() applies the glow variant if present. Both rely on prior Setup() call.

**Shutdown**: OGL_StopTextures() called at render session end.

**Special Effects**: Infravision and silhouette modes modify colors/pixels in-place via FindInfravision\* and FindSilhouette\* functions; activated by IsInfravisionActive() and SetInfravisionTint().

## External Dependencies
- **OpenGL**: GLuint, GLdouble, GLfloat, GLfloat\* (color and transformation data)
- **Game Engine Types**: shape_descriptor, bitmap_definition, RGBColor, rgb_color, ImageDescriptor, ImageDescriptorManager, OGL_TextureOptions, OGL_OpacType_Crisp, OGL_FIRST_PREMULT_ALPHA
- **Constants**: NUMBER_OF_OPENGL_BITMAP_SETS, MAXIMUM_SHADING_TABLE_INDEXES (defined elsewhere)
- **Endianness**: ALEPHONE_LITTLE_ENDIAN preprocessor flag for byte-order handling
