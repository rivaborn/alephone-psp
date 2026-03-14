# Source_Files/RenderMain/OGL_Setup.cpp

## File Purpose

OpenGL initialization and configuration management. Detects OpenGL presence, manages texture/model loading options, handles progress indication during resource loading, and parses XML configuration for rendering parameters like fog, textures, and landscape colors.

## Core Responsibilities

- Initialize and detect OpenGL availability on the host system (platform-specific implementations for Mac/SDL/Win32)
- Query OpenGL extensions and capabilities
- Provide default configuration values for texture filtering, resolution, and color formats
- Load/unload textures from disk with size constraints and mipmap generation
- Manage collections of models and images (load/unload by collection ID)
- Track and report progress during asset loading
- Parse XML configuration for fog parameters, texture options, and 3D models
- Validate and constrain texture dimensions (power-of-two, max size limits)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_FogData` | struct | Fog rendering parameters: color, depth, presence flags, landscape applicability |
| `OGL_Texture_Configure` | struct | Per-texture-type settings: near/far filters, resolution, color format, max size |
| `OGL_ConfigureData` | struct | Complete OpenGL configuration: texture settings, rendering flags, void color, landscape colors |
| `XML_FogParser` | class (extends `XML_ElementParser`) | Parses XML `<fog>` elements and updates fog configuration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `_OGL_IsPresent` | `bool` | static | Cached result of OpenGL availability check |
| `ogl_progress` | `int` | static | Current progress value during loading |
| `total_ogl_progress` | `int` | static | Total progress target |
| `show_ogl_progress` | `bool` | static | Whether progress indication is active |
| `last_update_tick` | `int32` | static | SDL tick timestamp of last progress update |
| `FogData[OGL_NUMBER_OF_FOG_TYPES]` | `OGL_FogData` array | static | Fog configuration for above/below-liquid scenarios |
| `DefaultLscpColors[4][2]` | `RGBColor` 2D array | static | Day/night/moon/space landscape colors (ground and sky) |
| `OriginalFogData` | `OGL_FogData*` | static | Backup pointer for XML parsing rollback |
| `FogParser` | `XML_FogParser` | static | Reusable fog XML element parser |
| `OpenGL_Parser` | `XML_ElementParser` | static | Root OpenGL XML element parser |

## Key Functions / Methods

### OGL_Initialize
- **Signature:** `bool OGL_Initialize()`
- **Purpose:** Detect OpenGL availability on the host system; initialize platform-specific OpenGL state
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö true if OpenGL is present and usable
- **Side effects:** Sets static `_OGL_IsPresent` flag; platform-specific initialization (Mac checks `aglChoosePixelFormat`, SDL assumes present)
- **Calls:** `aglChoosePixelFormat()` (Mac only), `setup_gl_extensions()` (Win32, currently commented out)
- **Notes:** Platform-conditional via preprocessor. Mac uses symbol resolution; SDL/Win32 assume availability.

### OGL_IsPresent
- **Signature:** `bool OGL_IsPresent()`
- **Purpose:** Query cached OpenGL availability state
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö value of `_OGL_IsPresent`
- **Side effects:** None
- **Calls:** None
- **Notes:** Simple getter; must call `OGL_Initialize()` first

### OGL_CheckExtension
- **Signature:** `bool OGL_CheckExtension(const std::string extension)`
- **Purpose:** Check if a specific OpenGL extension is supported
- **Inputs:** `extension` ΓÇö extension name (e.g., `"GL_ARB_texture_compression"`)
- **Outputs/Return:** `bool` ΓÇö true if extension is in the GL_EXTENSIONS string
- **Side effects:** Calls `glGetString(GL_EXTENSIONS)`
- **Calls:** `glGetString()`, `strcspn()`, `strncmp()`
- **Notes:** Only called if `HAVE_OPENGL` defined; parses space-separated extension list

### OGL_StartProgress
- **Signature:** `void OGL_StartProgress(int total_progress)`
- **Purpose:** Begin progress indication for long-running asset loads
- **Inputs:** `total_progress` ΓÇö the progress target value
- **Outputs/Return:** None
- **Side effects:** Initializes progress state; attempts to use `OGL_LoadScreen`, falls back to `open_progress_dialog()`; gets current SDL tick
- **Calls:** `OGL_LoadScreen::instance()`, `OGL_ClearScreen()`, `open_progress_dialog()`, `SDL_GetTicks()`
- **Notes:** Conditional on `HAVE_OPENGL`; load screen takes priority over fallback dialog

### OGL_ProgressCallback
- **Signature:** `void OGL_ProgressCallback(int delta_progress)`
- **Purpose:** Update progress indication incrementally
- **Inputs:** `delta_progress` ΓÇö amount to add to current progress
- **Outputs/Return:** None
- **Side effects:** Increments `ogl_progress`; updates display every ~33ms (throttled by SDL ticks)
- **Calls:** `SDL_GetTicks()`, `OGL_LoadScreen::instance()`, `draw_progress_bar()`
- **Notes:** Throttled to avoid excessive redraws; uses either load screen or progress bar dialog

### OGL_StopProgress
- **Signature:** `void OGL_StopProgress()`
- **Purpose:** Finish progress indication and clean up UI
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `show_ogl_progress` flag; stops load screen or closes dialog
- **Calls:** `OGL_LoadScreen::instance()`, `close_progress_dialog()`
- **Notes:** Conditional on `HAVE_OPENGL`

### OGL_SetDefaults
- **Signature:** `void OGL_SetDefaults(OGL_ConfigureData& Data)`
- **Purpose:** Initialize configuration structure with engine-recommended defaults
- **Inputs:** `Data` ΓÇö reference to configuration struct to populate
- **Outputs/Return:** None (modifies `Data` in-place)
- **Side effects:** Writes default filter/resolution/format values; sets platform-specific feature flags (SDL vs Mac Carbon vs generic); initializes anisotropy, multisampling, void color, and landscape colors
- **Calls:** None
- **Notes:** Different flag sets per platform; anisotropy and multisampling default to off; GeForceFix defaults to false

### OGL_TextureOptionsBase::Load
- **Signature:** `void OGL_TextureOptionsBase::Load()`
- **Purpose:** Load texture and optional mask/glow from disk files; apply constraints and preprocessing
- **Inputs:** None (uses member variables: `NormalColors`, `NormalMask`, `GlowColors`, `GlowMask`, `Type`)
- **Outputs/Return:** None (updates `NormalImg`, `GlowImg`, `actual_width`, `actual_height`)
- **Side effects:** File I/O; allocates GPU textures via `ImageLoader`; may minify textures; validates constraints (glow only if normal present, same dimensions)
- **Calls:** `glGetIntegerv(GL_MAX_TEXTURE_SIZE)`, `OGL_CheckExtension()`, `Get_OGL_ConfigureData()`, `NormalImg.LoadFromFile()`, `NormalImg.Minify()`, `GlowImg.Clear()`, `Color_SetArray()`
- **Notes:** Flags conditionally set mipmaps, power-of-two resizing, and DXtc compression; glow texture removed if dimensions don't match normal; early return if normal image missing

### OGL_TextureOptionsBase::Unload
- **Signature:** `void OGL_TextureOptionsBase::Unload()`
- **Purpose:** Release texture resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `NormalImg` and `GlowImg`
- **Calls:** `NormalImg.Clear()`, `GlowImg.Clear()`
- **Notes:** Simple cleanup routine

### OGL_TextureOptionsBase::GetMaxSize
- **Signature:** `int OGL_TextureOptionsBase::GetMaxSize()`
- **Purpose:** Query the maximum texture size for this texture type
- **Inputs:** None (uses member `Type`)
- **Outputs/Return:** `int` ΓÇö max size from config, or 0 (unlimited)
- **Side effects:** None
- **Calls:** `Get_OGL_ConfigureData()`
- **Notes:** Bounds-checked on `Type`; returns 0 if out of range

### OGL_LoadModelsImages / OGL_UnloadModelsImages
- **Signature:** `void OGL_LoadModelsImages(short Collection)` / `void OGL_UnloadModelsImages(short Collection)`
- **Purpose:** Manage asset loading/unloading for a collection (walls, sprites, models, skins)
- **Inputs:** `Collection` ΓÇö collection ID (0ΓÇôMAXIMUM_COLLECTIONS)
- **Outputs/Return:** None
- **Side effects:** Calls texture/model load/unload functions; respects `OGL_Flag_3D_Models` flag
- **Calls:** `OGL_LoadTextures()`, `OGL_UnloadTextures()`, `OGL_LoadModels()`, `OGL_UnloadModels()`, `Get_OGL_ConfigureData()`, `TEST_FLAG()`
- **Notes:** Asserts collection is in range; models loaded only if flag enabled; both load and unload variants, plus no-op stubs when `HAVE_OPENGL` not defined

### XML_FogParser::Start
- **Signature:** `bool XML_FogParser::Start()`
- **Purpose:** Initialize fog XML parsing; back up current fog data
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö always true
- **Side effects:** Allocates and copies `FogData` to `OriginalFogData` on first call; resets `IsPresent` flags and `Type`
- **Calls:** `malloc()`, `assert()`
- **Notes:** Backup is only done once (checked via `OriginalFogData` null test)

### XML_FogParser::HandleAttribute
- **Signature:** `bool XML_FogParser::HandleAttribute(const char *Tag, const char *Value)`
- **Purpose:** Parse XML attribute (on, depth, landscapes, type) and store value
- **Inputs:** `Tag` ΓÇö attribute name; `Value` ΓÇö attribute value string
- **Outputs/Return:** `bool` ΓÇö true if successfully parsed, false otherwise
- **Side effects:** Sets member variables (`FogPresent`, `Depth`, `AffectsLandscapes`, `Type`, `IsPresent` flags)
- **Calls:** `StringsEqual()`, `ReadBooleanValueAsBool()`, `ReadFloatValue()`, `ReadBoundedInt16Value()`, `UnrecognizedTag()`
- **Notes:** Supports "on", "depth", "landscapes", "type" attributes; type is bounds-checked; unrecognized tags trigger warning

### XML_FogParser::AttributesDone
- **Signature:** `bool XML_FogParser::AttributesDone()`
- **Purpose:** Apply parsed attribute values to fog data
- **Inputs:** None (uses member variables and `IsPresent` flags)
- **Outputs/Return:** `bool` ΓÇö always true
- **Side effects:** Updates `FogData[Type]` with parsed values (if present flag set)
- **Calls:** `Color_SetArray()`
- **Notes:** Only updates fields that were explicitly set in XML

### XML_FogParser::ResetValues
- **Signature:** `bool XML_FogParser::ResetValues()`
- **Purpose:** Restore fog data to original values (undo XML parse)
- **Inputs:** None
- **Outputs/Return:** `bool` ΓÇö always true
- **Side effects:** Copies `OriginalFogData` back to `FogData`; frees and nulls `OriginalFogData`
- **Calls:** `free()`
- **Notes:** Safe to call if backup was never made

### OpenGL_GetParser
- **Signature:** `XML_ElementParser *OpenGL_GetParser()`
- **Purpose:** Build and return the root XML parser for OpenGL configuration
- **Inputs:** None
- **Outputs/Return:** `XML_ElementParser*` ΓÇö pointer to `OpenGL_Parser`
- **Side effects:** Adds child parsers (texture, model, fog, color)
- **Calls:** `OpenGL_Parser.AddChild()`, `TextureOptions_GetParser()`, `TO_Clear_GetParser()`, `ModelData_GetParser()`, `Mdl_Clear_GetParser()`, `Color_GetParser()`
- **Notes:** Calls are conditional on `HAVE_OPENGL` for texture/model parsers; fog and color parsers always added; returns same parser on each call (subsequent calls add duplicate children)

## Control Flow Notes

This file is primarily **initialization** and **configuration-loading** phase. `OGL_Initialize()` is called early to detect GPU capability. `OGL_SetDefaults()` establishes baseline settings. During engine startup or when loading levels, `OpenGL_GetParser()` is used to parse XML configuration (textures, fog, models). `OGL_LoadModelsImages()` is called per-collection when that collection's assets are needed (likely in level setup). Progress callbacks are used during long asset-load operations (textures, models) to provide user feedback. The fog, texture, and model parsers persist as static singletons and are reused across multiple XML documents.

## External Dependencies

- **OpenGL:** `glGetString()`, `glGetIntegerv()` (gl.h, agl.h, GL/gl.h depending on platform)
- **SDL:** `SDL_GetTicks()` (SDL.h)
- **Game engine internals (defined elsewhere):**
  - `OGL_LoadScreen`, `OGL_ClearScreen()` (OGL_LoadScreen.h, OGL_Blitter.h)
  - `open_progress_dialog()`, `close_progress_dialog()`, `draw_progress_bar()` (progress.h)
  - `OGL_LoadTextures()`, `OGL_UnloadTextures()`, `OGL_CountTextures()` (OGL_Textures.cpp)
  - `OGL_LoadModels()`, `OGL_UnloadModels()`, `OGL_CountModels()` (OGL_Models.cpp)
  - `Get_OGL_ConfigureData()`, texture/model parser factories (OGL_Subst_Texture_Def.h, OGL_Model_Def.h)
  - `Color_SetArray()`, `Color_GetParser()` (ColorParser.h)
  - `XML_ElementParser`, XML utility functions (XML_ElementParser.h)
  - `StringsEqual()`, `ReadBooleanValueAsBool()`, `ReadFloatValue()`, `ReadBoundedInt16Value()` (XML parsing utilities, cseries.h)
  - `TEST_FLAG()`, `MIN()`, `GetMemberWithBounds()` (cseries.h macros)
  - Collection constants `MAXIMUM_COLLECTIONS` (shape_descriptors.h)
