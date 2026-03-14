# Source_Files/RenderMain/AnimatedTextures.cpp
## File Purpose
Implements animated wall textures for the Aleph One game engine by managing texture frame sequences and their timing. Provides XML-configurable animation sequences that translate texture descriptors at runtime, allowing seamless texture animation in level rendering.

## Core Responsibilities
- Manage animation sequences as frame lists with independent timing (advance/reverse)
- Track animation state per sequence (current frame and tick phase)
- Translate texture descriptors to animated frames during rendering
- Parse XML configuration to define animation sequences per collection
- Maintain per-collection animation registry for efficient lookup during frame rendering

## External Dependencies
- **Headers:** `<vector>`, `cseries.h`, `AnimatedTextures.h`, `interface.h`
- **External symbols (defined elsewhere):**
  - `shape_descriptor`, `GET_DESCRIPTOR_SHAPE`, `GET_DESCRIPTOR_COLLECTION`, `GET_COLLECTION`, `GET_COLLECTION_CLUT`, `BUILD_DESCRIPTOR`, `BUILD_COLLECTION`, `UNONE` ΓÇô from shape_descriptors.h
  - `NUMBER_OF_COLLECTIONS` ΓÇô constant
  - `get_number_of_collection_frames()` ΓÇô from interface.h
  - `XML_ElementParser` ΓÇô base class for XML parsing framework
  - `StringsEqual()`, `ReadBoundedInt16Value()`, `ReadInt16Value()`, `ReadUInt16Value()` ΓÇô XML utility functions (likely in cseries or XML module)

# Source_Files/RenderMain/AnimatedTextures.h
## File Purpose
Provides the interface for animated texture management in the Aleph One rendering pipeline. Handles per-frame animation updates, texture descriptor translation (mapping static textures to animated variants), and XML-based configuration loading.

## Core Responsibilities
- Update animated texture states each frame
- Translate shape descriptors to animated equivalents during rendering
- Provide XML parser for loading animated texture configurations
- Manage the mapping between static and animated texture IDs

## External Dependencies
- **shape_descriptors.h** ΓÇö `shape_descriptor` typedef; defines descriptor bit layout and collection enums
- **XML_ElementParser.h** ΓÇö `XML_ElementParser` base class for parsing configuration

# Source_Files/RenderMain/collection_definition.h
## File Purpose
Defines core data structures for asset collections in the rendering system. Collections are containers holding shape animations, individual frames/bitmaps, and color lookup tables (CLUTs) used throughout the game engine.

## Core Responsibilities
- Define `collection_definition` structure as the primary container for all shape and bitmap assets
- Specify `high_level_shape_definition` for animation metadata (views, frames, transfer modes, sounds)
- Specify `low_level_shape_definition` for individual bitmap frames with rendering parameters (mirroring, lighting, origin/keypoint positions)
- Specify `rgb_color_value` structure for color table entries
- Define collection type enumeration (_wall_collection, _object_collection, _interface_collection, _scenery_collection)
- Provide versioning, size constants, and bit-flag definitions for rendering parameters

## External Dependencies
- **Standard library**: `<vector>` (used for dynamic arrays in collection_definition)
- **Forward declarations**: `struct bitmap_definition`, `struct high_level_shape_definition`, `struct low_level_shape_definition`, `struct rgb_color_value`
- **Referenced but not defined**: `_fixed` type (likely defined in a separate header), `shape_animation_data` (mentioned in comment as interface.h definition)

---

**Notes:**
- `high_level_shape_definition.low_level_shape_indexes[1]` is a flexible array member; actual count depends on `number_of_views ├ù frames_per_view` (see interface.h/shape_animation_data).
- `pixels_to_world` appears in both collection_definition and high_level_shape_definition, allowing per-shape override of coordinate scaling.
- Bit flags in low_level_shape_definition encode rendering intent (mirroring, keypoint visibility) and lighting requirements in a single uint16.

# Source_Files/RenderMain/Crosshairs.h
## File Purpose
Interface header for the crosshair rendering system in Aleph One. Defines the data structure for crosshair configuration and provides functions to render, configure, and toggle crosshairs in the game viewport.

## Core Responsibilities
- Define `CrosshairData` structure for storing crosshair visual properties (color, thickness, opacity, shape)
- Provide configuration dialog for user customization of crosshair appearance
- Manage crosshair active/inactive state
- Render crosshairs to screen via platform-specific graphics APIs (QuickDraw for Mac, SDL for cross-platform)
- Retrieve crosshair configuration from persistent preferences

## External Dependencies
- `RGBColor` type (defined elsewhere)
- QuickDraw types (`Rect`, `GrafPtr`) for Mac builds
- SDL types (`SDL_Surface`) for SDL builds
- `PlayerDialogs.c` ΓÇö implementation of configuration dialog
- `preferences.c` ΓÇö storage and retrieval of crosshair configuration

# Source_Files/RenderMain/Crosshairs_SDL.cpp
## File Purpose
SDL-based implementation for rendering game crosshairs. Provides state management (active/inactive) and rendering logic for two crosshair shapes: traditional plus-sign and circular (octagon approximation).

## Core Responsibilities
- Manage crosshairs active/inactive toggle state
- Render plus-sign crosshairs via four filled rectangles (left/right/top/bottom arms)
- Render circular crosshairs as octagon using 12 line segments (4 quadrants ├ù 3 segments each)
- Color mapping from game CrosshairData to SDL pixel format
- Center crosshairs on render surface

## External Dependencies
- **SDL**: SDL_Surface, SDL_Rect, SDL_FillRect(), SDL_MapRGB()
- **Crosshairs.h**: CrosshairData struct, GetCrosshairData(), Crosshairs_IsActive(), Crosshairs_SetActive()
- **screen_drawing.h**: draw_line(SDL_Surface*, world_point2d*, world_point2d*, uint32, int)
- **world.h**: world_point2d struct
- **cseries.h**: uint32, int16, short type definitions

# Source_Files/RenderMain/DDS.h
## File Purpose
Defines DirectDraw Surface (DDS) file format structures and constants for parsing DDS image files. Part of the Aleph One game engine's rendering subsystem. Based on Microsoft's DirectX 9 DDS file format reference.

## Core Responsibilities
- Define DDS surface descriptor flags (`DDSD_*`) for validating which header fields are present
- Define pixel format flags (`DDPF_*`) to identify color formats (RGB, FourCC, alpha)
- Define surface capability flags (`DDSCAPS_*`, `DDSCAPS2_*`) for texture properties (mipmaps, cubemaps, volumes)
- Declare `DDSURFACEDESC2` structure representing the complete DDS file header
- Provide a portable definition independent of native DirectDraw headers (guarded by `__DDRAW_INCLUDED__`)

## External Dependencies
- `cstypes.h` ΓÇö provides platform-independent `uint32` type
- Conditional guard `__DDRAW_INCLUDED__` to avoid conflicts with native DirectDraw SDK headers


# Source_Files/RenderMain/ImageLoader.h
## File Purpose
Defines the `ImageDescriptor` class for managing loaded image data and texture loading operations. Supports multiple pixel formats (RGBA8, DXTC compression), mipmap hierarchies, and alpha channel operations. Provides a copy-on-edit template wrapper for efficient resource management.

## Core Responsibilities
- **Image Data Management**: Holds pixel buffers, dimensions, and metadata (format, scaling)
- **Format Support**: RGBA8 and DirectX Texture Compression (DXTC1/3/5) formats
- **Mipmap Management**: Stores and accesses multiple detail levels with size calculation
- **Format Conversion**: Convert between RGBA and DXTC3 formats; premultiply alpha
- **File Loading**: Load images from disk (DDS format, with optional mipmap levels)
- **Resource Resizing**: Reallocate and resize image data; minify/reduce resolution
- **Copy-on-Edit Pattern**: Template for efficient copy-only-when-modified semantics

## External Dependencies
- **DDS.h:** `DDSURFACEDESC2` struct (DirectDraw Surface format descriptor)
- **cseries.h:** Platform abstractions, type definitions
- **FileHandler.h:** `FileSpecifier`, `OpenedFile` classes for file I/O
- **\<vector\>:** STL vector (included but not directly used in this header; possibly for implementation)
- **uint32, uint8, uint16** ΓÇö Fixed-width integer types (assumed from cstypes.h)

# Source_Files/RenderMain/ImageLoader_SDL.cpp
## File Purpose
SDL-based image file loader for the Aleph One game engine. Loads image files (BMP, PNG, JPEG, etc.) via SDL_image and converts them to 32-bit RGBA format suitable for OpenGL rendering. Supports loading both color data and separate opacity (alpha) channels.

## Core Responsibilities
- Load image files from disk using SDL_image (or SDL_LoadBMP as fallback)
- Convert arbitrary image formats to OpenGL-friendly 32-bit RGBA surface
- Handle platform-specific endianness for RGBA pixel layout
- Optionally resize images to power-of-two dimensions
- Support dual-channel loading: color (RGB) and opacity (grayscale-to-alpha)
- Manage SDL surface allocation and cleanup
- Validate opacity channel matches previously-loaded color dimensions

## External Dependencies
- **Includes:** `ImageLoader.h` (class definition, constants), `FileHandler.h` (FileSpecifier), `<SDL_image.h>` (optional, conditional on `HAVE_SDL_IMAGE_H`)
- **External functions:**
  - `NextPowerOfTwo(int)`: Rounds value to next power of 2 (defined elsewhere)
  - `PIN(int val, int min, int max)`: Clamps value to range (likely macro)
  - `vassert()`, `csprintf()`: Assertion and formatted string (likely in cseries.h)
  - `temporary`: Global or thread-local string buffer for error messages
- **SDL API (linked at runtime):**
  - `IMG_Load(const char *file)`: Load any supported image format
  - `SDL_LoadBMP(const char *file)`: Load BMP (fallback if SDL_image unavailable)
  - `SDL_CreateRGBSurface(flags, w, h, depth, Rmask, Gmask, Bmask, Amask)`: Create RGBA surface
  - `SDL_SetAlpha(surface, flags, alpha)`: Disable alpha blending
  - `SDL_BlitSurface(src, srcrect, dst, dstrect)`: Copy with format conversion
  - `SDL_FreeSurface(surface)`: Deallocate surface memory

# Source_Files/RenderMain/ImageLoader_Shared.cpp
## File Purpose
Implements image loading and decompression for DDS (DirectDraw Surface) texture files, with support for DXTC compressed formats (DXTC1/3/5) and RGBA8. Provides mipmap handling, format conversion, and DXTC decompression (adapted from DevIL).

## Core Responsibilities
- Load and parse DDS file headers and texture data
- Handle mipmap chains (load, skip, or resize)
- Decompress DXTC1, DXTC3, and DXTC5 compressed formats to RGBA8
- Convert between image formats (DXTCΓåöRGBA, DXTC1ΓåöDXTC3)
- Resize images and calculate mipmap level sizes
- Apply power-of-two resizing constraints
- Premultiply alpha channel for correct blending
- Handle endianness conversions for binary data

## External Dependencies
- **SDL** (`SDL.h`, `SDL_endian.h`): Surface creation and pixel blitting for format conversion.
- **OpenGL** (`gl.h`, `glu.h`, `OGL_Setup.h`): Conditional support for image scaling via `gluScaleImage` when GL is active.
- **Custom streams** (`AStream.h`): `AIStreamLE` for little-endian binary parsing of DDS header.
- **Type definitions** (`cstypes.h`): `uint32`, `uint16`, `uint8` integer types; `FOUR_CHARS_TO_INT` macro.
- **DDS structures** (`DDS.h`): `DDSURFACEDESC2` and surface descriptor flags (DDSD_*, DDPF_*, DDSCAPS_*, DDSCAPS2_*).
- **File handling** (`ImageLoader.h`, `FileHandler.h` via OpenedFile/FileSpecifier): File I/O abstraction.
- **Math**: Standard C++ `<cmath>` for `log2`, `floor`, `max`.

# Source_Files/RenderMain/low_level_textures.h
## File Purpose
Low-level templated texture rasterization for the Marathon-like game engine. Provides functions to render horizontal and vertical textured polygon scanlines directly to a bitmap surface, with support for multiple pixel depths (8/16/32-bit), alpha blending modes, transparency, tinting, and dithering.

## Core Responsibilities
- Pixel blending primitives: averaging and alpha-blending for different bit depths
- Horizontal scanline texture mapping with per-pixel source address calculation
- Optimized landscape texture rendering with fixed downshift calculations
- Vertical column texture mapping with 4-pixel-wide SIMD-style optimization
- Transparency-aware pixel writes with configurable alpha blend modes
- Color tinting operations via lookup tables
- Stochastic dithering/randomization effects on textured surfaces
- Support for per-pixel opacity tables in alpha blending

## External Dependencies
- **SDL library**: `SDL_Surface`, `SDL_PixelFormat` (pixel format metadata, framebuffer access)
- **Macro**: `FIXED_INTEGERAL_PART()` ΓÇö integer part of a fixed-point value (defined elsewhere)
- **Function**: `NextLowerExponent()` ΓÇö logΓéé of next lower power of 2 (defined elsewhere)
- **Types**: `bitmap_definition`, `view_data`, `_horizontal_polygon_line_data`, `_vertical_polygon_data`, `_vertical_polygon_line_data`, `tint_table*` (all defined elsewhere in engine headers)
- **Global externs**: `world_pixels`, `texture_random_seed`, `number_of_shading_tables`

# Source_Files/RenderMain/OGL_Faders.cpp
## File Purpose
Implements OpenGL-based visual fade effects (color overlays, tinting, effects) rendered each frame. Supports multiple blend modes for game effects like explosions, underwater/lava tinting, damage feedback, and visual randomness.

## Core Responsibilities
- Manages a queue of fader effects (liquid environment tints and general effects)
- Implements six distinct fader blend modes using OpenGL primitives
- Renders fader overlays using vertex arrays and configurable blend functions
- Provides fader queue access and activation checks
- Handles both standard blending and logic-op color manipulation

## External Dependencies
- **OpenGL**: `GL/gl.h` (platform-conditional includes for Windows, macOS, Linux)
- **Engine faders**: `fades.h` (fade type enums, fader queue constants)
- **Rendering**: `render.h`, `OGL_Render.h` (context checks, configuration access)
- **Random**: `Random.h` (GM_Random pseudo-RNG)
- **Utilities**: `cseries.h` (macros, type definitions), `OGL_Faders.h` (header)
- **Defined elsewhere**: `OGL_IsActive()`, `Get_OGL_ConfigureData()`, `TEST_FLAG()` macro

# Source_Files/RenderMain/OGL_Faders.h
## File Purpose
Declares the OpenGL renderer's fader system interface, which manages fade-in/fade-out screen transitions with color and transparency. Part of Aleph One's rendering pipeline to handle visual fading effects.

## Core Responsibilities
- Define the `OGL_Fader` data structure for storing fade parameters
- Declare query functions to check fader state and access fader queue entries
- Declare the main fader rendering function that applies fades to a rectangular viewport region
- Define fader queue type categories (liquid vs. other)

## External Dependencies
- OpenGL (implicit; referenced in file comments)
- Fader queue management (implementation elsewhere; accessed via `GetOGL_FaderQueueEntry()`)
- Type definition for `NONE` constant (defined elsewhere)

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

## External Dependencies

- **Includes:**  
  `cseries.h` (common utility/types), `OGL_Model_Def.h` (header), `OGL_Setup.h` (config access), `cmath` (trig), `Dim3_Loader.h`, `StudioLoader.h`, `WavefrontLoader.h`, `QD3D_Loader.h` (model loaders)

- **Defined elsewhere (used here):**  
  `OGL_TextureOptionsBase` (base class for OGL_SkinData), `Model3D` (3D geometry container), `XML_ElementParser` (XML parsing base), `FileSpecifier` (file I/O), `OGL_SkinData::Load/Unload`, `OGL_SkinData::GetMaxSize`, `Get_OGL_ConfigureData()`, `OGL_ProgressCallback()`, `objlist_copy`, `objlist_clear`, `objlist_set`, `StringsEqual`, `ReadBoundedInt16Value`, `ReadFloatValue`, `ReadInt16Value`, OpenGL functions (`glGenTextures`, `glBindTexture`, `glDeleteTextures`)

- **Constants/Enums (defined elsewhere):**  
  `NUMBER_OF_COLLECTIONS`, `MAXIMUM_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `NUMBER_OF_OPENGL_BITMAP_SETS`, `OGL_NUMBER_OF_OPACITY_TYPES`, `OGL_NUMBER_OF_BLEND_TYPES`, `Model3D::NUMBER_OF_NORMAL_TYPES`, `NUMBER_OF_MODEL_LIGHT_TYPES`, `ALL_CLUTS`, `SILHOUETTE_BITMAP_SET`, `NONE`

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

## External Dependencies
- **OGL_Texture_Def.h**: `OGL_TextureOptionsBase` (base class for texture options)
- **Model3D.h**: `Model3D` struct (geometry, bones, frames, sequences)
- **XML_ElementParser.h**: `XML_ElementParser` (XML parsing framework)
- **OpenGL headers** (platform-conditional: `<OpenGL/gl.h>`, `<GL/gl.h>`, etc.)
- **STL**: `<vector>`
- **cstypes.h** (via XML_ElementParser.h, for integer types)

# Source_Files/RenderMain/OGL_Render.cpp
## File Purpose
OpenGL renderer implementation for the Marathon/Aleph One game engine. Manages OpenGL context lifecycle, coordinate transformations, matrix setup, texture/material blending, and 2D/3D rendering. Bridges OpenGL API with engine-level rendering requests for walls, objects, text, and UI elements.

## Core Responsibilities
- OpenGL context initialization, management, and teardown
- Coordinate system transformations (Marathon world ΓåÆ OpenGL eye space)
- Projection matrix selection and management (3D depth vs. flat screen)
- Texture blending and alpha-test configuration for materials
- Static effect (TV noise) rendering via stipple patterns or stencil buffer
- Shader callbacks for standard, glowing, and special-effect textures
- Per-vertex lighting calculations for 3D models
- Crosshair and text rendering in screen space
- 2D graphics buffer management (Mac platform)
- Platform-specific window attachment and context management

## External Dependencies
**Includes/Imports:**
- STL: `<vector>`, `<set>`, `<algorithm>`
- Standard C: `<string.h>`, `<stdlib.h>`, `<math.h>`
- Engine: `cseries.h`, `world.h`, `shell.h`, `preferences.h`, `interface.h`, `render.h`, `map.h`, `player.h`
- Rendering: `OGL_Render.h`, `OGL_Textures.h`, `AnimatedTextures.h`, `Crosshairs.h`, `VecOps.h`, `Random.h`, `ViewControl.h`, `OGL_Faders.h`, `ModelRenderer.h`, `Logging.h`
- OpenGL: `<OpenGL/gl.h>`, `<OpenGL/glu.h>` (macOS) or `<GL/gl.h>`, `<GL/glu.h>` (other); `<AGL/agl.h>` (Mac-specific)
- Platform: `OGL_Win32.h` (Windows), `my32bqd.h` (Mac)

**Symbols defined elsewhere (not visible here):**
- `OGL_IsPresent()`, `OGL_StartProgress()`, `OGL_ProgressCallback()`, `OGL_StopProgress()` ΓÇô OpenGL lifecycle
- `Get_OGL_ConfigureData()`, `TEST_FLAG()`, `SET_FLAG()` ΓÇô Configuration and bit-field macros
- `load_replacement_collections()`, `count_replacement_collections()` ΓÇô Resource management
- `OGL_StartTextures()`, `OGL_ResetTextures()` ΓÇô Texture subsystem
- `PreloadTextures()`, `PreloadWallTexture()` ΓÇô Texture preloading
- `GetOnScreenFont()`, `OGL_ResetMapFonts()`, `OGL_ResetHUDFonts()` ΓÇô Font system
- `OGL_ResetModelSkins()`, `LoadModelSkin()` ΓÇô Model skin management
- `GetCrosshairData()` ΓÇô Crosshair parameters
- `FindShadingColor()` ΓÇô Lighting color lookup table
- `setup_gl_extensions()` ΓÇô Platform GL extension initialization

# Source_Files/RenderMain/OGL_Render.h
## File Purpose
Public interface for OpenGL 3D rendering in the Aleph One game engine (Marathon port). Declares functions to manage OpenGL contexts, rendering state, and draw game objects (walls, sprites, text, HUD). Separated from OGL_Control because this header is used by rendering code.

## Core Responsibilities
- OpenGL context lifecycle (initialization, activation checks, startup/shutdown)
- Rendering state management (viewport bounds, view parameters, buffer swapping)
- Drawing 3D geometry (walls, sprites, crosshairs)
- Drawing 2D overlays (text, HUD, status bar, overhead map via OGL_Copy2D)
- Foreground object rendering (weapons in hand)
- Special effects (infravision tinting, render modes)
- Cross-platform abstraction (Mac-specific variants handled with `#ifdef mac`)

## External Dependencies
- **Includes:** `OGL_Setup.h` (configuration, data structures, extension checks)
- **Opaque types:** `view_data`, `polygon_definition`, `rectangle_definition` (defined elsewhere)
- **Platform types:** `CGrafPtr`, `GWorldPtr`, `Rect` (Mac Toolbox / Quickdraw; conditional on `#ifdef mac`)
- **Assumed symbols:** OpenGL context management, GPU draw calls (implementation in corresponding .cpp file)

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

## External Dependencies
- `OGL_TextureOptionsBase` ΓÇö base class with opacity, blending, image/mask paths, premultiply flags (defined in `OGL_Texture_Def.h`)
- `XML_ElementParser` ΓÇö base class for XML parsing framework
- STL: `<deque>`, `vector` for dynamic collections
- Undefined here: `OGL_ProgressCallback()`, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `ReadBooleanValueAsBool()`, `ReadString()`, `StringsEqual()`, `ALL_CLUTS`, `NONE`, `SILHOUETTE_BITMAP_SET`, `OGL_NUMBER_OF_OPACITY_TYPES`, `OGL_NUMBER_OF_BLEND_TYPES`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`, `UNONE`, `TEST_FLAG()`, `SET_FLAG()`

# Source_Files/RenderMain/OGL_Subst_Texture_Def.h
## File Purpose
Defines substitute texture options and management functions for OpenGL rendering of walls and sprites in the Aleph One engine. This header specifies how custom textures (loaded from image files) replace the original in-game graphics, including positioning and transparency behavior.

## Core Responsibilities
- Define extended texture options (`OGL_TextureOptions`) for sprite and wall substitutions
- Provide sprite positioning parameters (scaling, offset calculation)
- Expose texture collection management (load, unload, count)
- Support XML-based texture configuration parsing
- Handle image positioning calculation relative to original bitmaps

## External Dependencies
- `OGL_Texture_Def.h`: Base texture options struct and enums (opacity types, blend types, ImageDescriptor)
- `XML_ElementParser.h`: XML parsing framework for configuration
- Conditional compilation: `#ifdef HAVE_OPENGL` (entire file)

# Source_Files/RenderMain/OGL_Texture_Def.h
## File Purpose

Defines base structures and enums for OpenGL texture configuration, supporting wall/sprite substitutions and model skins. Accommodates OpenGL 1.1's limitations on indexed-color rendering by providing separate bitmap sets for different color tables, infravision, and silhouettes. Manages texture opacity, blending modes, and image loading metadata.

## Core Responsibilities

- Define texture opacity types (crisp, flat, average-based, max-based)
- Define texture blending modes (crossfade, additive, with optional premultiplication)
- Define bitmap set enumeration for color tables, infravision, and silhouettes
- Provide base struct `OGL_TextureOptionsBase` for shared texture configuration
- Declare texture load/unload lifecycle methods
- Track normal and glow-mapped image descriptors and their blend settings

## External Dependencies

- **Standard Library:** `<vector>` (std::vector for file paths)
- **Internal Headers:**
  - `shape_descriptors.h` ΓÇö defines `shape_descriptor`, collection/CLUT constants (`MAXIMUM_CLUTS_PER_COLLECTION`)
  - `ImageLoader.h` ΓÇö defines `ImageDescriptor` class for holding loaded pixel data
- **Conditional:** Entire file guarded by `#ifdef HAVE_OPENGL`

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

## External Dependencies
- **OpenGL:** `glGenTextures`, `glDeleteTextures`, `glBindTexture`, `glTexImage2D`, `glCompressedTexImage2DARB`, `gluBuild2DMipmaps`, `glGetIntegerv`, `glTexParameteri`, `glTexEnvi`, `glMatrixMode`, `glLoadIdentity`, `glRotatef`, `glScalef`, `glTranslatef`
- **Marathon Engine:** `get_bitmap_index`, `get_number_of_collection_bitmaps`, `is_collection_present`, `GET_DESCRIPTOR_COLLECTION/SHAPE`, `GET_COLLECTION_CLUT`, `OGL_GetTextureOptions`, `Get_OGL_ConfigureData`, `OGL_CheckExtension`, `OGL_IsActive`, `OGL_ResetModelSkins`
- **Image/Texture:** `ImageDescriptor`, `ImageDescriptorManager`, `OGL_TextureOptions`
- **SDL:** `SDL_SwapLE16`, `SDL_Color` (for endianness, color type)

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

## External Dependencies
- **OpenGL**: GLuint, GLdouble, GLfloat, GLfloat\* (color and transformation data)
- **Game Engine Types**: shape_descriptor, bitmap_definition, RGBColor, rgb_color, ImageDescriptor, ImageDescriptorManager, OGL_TextureOptions, OGL_OpacType_Crisp, OGL_FIRST_PREMULT_ALPHA
- **Constants**: NUMBER_OF_OPENGL_BITMAP_SETS, MAXIMUM_SHADING_TABLE_INDEXES (defined elsewhere)
- **Endianness**: ALEPHONE_LITTLE_ENDIAN preprocessor flag for byte-order handling

# Source_Files/RenderMain/OGL_Win32.cpp
## File Purpose
Initializes OpenGL ARB extensions on Windows by dynamically loading extension function pointers via SDL. Detects and enables multitexturing capability and compressed texture support for the renderer.

## Core Responsibilities
- Dynamically load OpenGL extension function pointers using `SDL_GL_GetProcAddress`
- Detect availability of ARB multitexturing extensions
- Initialize compressed texture extension pointer
- Set global flag to indicate multitexture support status
- Report missing extensions to stderr

## External Dependencies
- **windows.h** ΓÇö Windows API (platform target)
- **GL/gl.h, GL/glext.h** ΓÇö OpenGL core and extension definitions
- **SDL.h** ΓÇö SDL library for cross-platform GL proc address loading
- **OGL_Win32.h** ΓÇö Local header defining function pointer types and extern declarations

# Source_Files/RenderMain/OGL_Win32.h
## File Purpose
Windows-specific header that provides dynamic function pointers for OpenGL ARB extensions (multitexturing and compressed textures). Supports runtime discovery and binding of these optional extensions through conditional compilation patterns.

## Core Responsibilities
- Define function pointer types for OpenGL ARB extensions
- Declare and conditionally export global function pointers
- Track multitexture support availability via a flag
- Provide macros that map extension function calls to dynamically resolved pointers
- Separate implementation initialization (`__OGL_Win32_cpp__`) from consumer code

## External Dependencies
- `<GL/gl.h>` ΓÇö OpenGL core headers
- `<GL/glext.h>` ΓÇö OpenGL extensions; provides `PFNGLCOMPRESSEDTEXIMAGE2DARBPROC`
- Implementation file: `OGL_Win32.cpp` (defines `setup_gl_extensions()`)

# Source_Files/RenderMain/Rasterizer.h
## File Purpose
Abstract base class defining the interface for rasterizer implementations (software, OpenGL, etc.). Serves as the primary abstraction layer for rendering operations within the game's 3D/2D rendering pipeline. Subclasses provide concrete implementations for different rendering backends.

## Core Responsibilities
- Define the virtual interface for view setup and perspective configuration
- Manage rendering lifecycle (Begin/End boundaries)
- Provide texture rasterization for polygons (horizontal and vertical) and rectangles
- Handle foreground object rendering (weapons in hand) with optional reflection
- Allow backend-specific implementations to override all operations

## External Dependencies
- `#include "render.h"` ΓÇö provides `view_data`, `polygon_definition`, `rectangle_definition`, `bitmap_definition`
- `#include "OGL_Render.h"` (conditional `HAVE_OPENGL`) ΓÇö OpenGL-specific definitions, included but not used in this header
- Standard C++ virtual method mechanism

# Source_Files/RenderMain/Rasterizer_OGL.h
## File Purpose
OpenGL implementation of the rasterizer interface. Provides a concrete class that delegates rendering operations to underlying OpenGL (OGL_*) functions while maintaining the abstract rasterizer API contract.

## Core Responsibilities
- Implement the abstract `RasterizerClass` interface for OpenGL-based rendering
- Manage view and projection state for OpenGL rendering context
- Delegate rendering of textured polygons and rectangles to OGL functions
- Support foreground rendering for UI elements (weapons in hand, etc.)
- Provide frame delimiters (Begin/End) for OpenGL rendering passes

## External Dependencies
- **Base class**: `RasterizerClass` (Rasterizer.h) ΓÇö abstract rendering interface
- **OpenGL functions** (defined elsewhere): `OGL_SetView()`, `OGL_SetForeground()`, `OGL_SetForegroundView()`, `OGL_StartMain()`, `OGL_EndMain()`, `OGL_RenderWall()`, `OGL_RenderSprite()`
- **Data structures** (defined elsewhere): `view_data`, `polygon_definition`, `rectangle_definition`

# Source_Files/RenderMain/Rasterizer_SW.h
## File Purpose
Defines the software rasterizer implementation class (`Rasterizer_SW_Class`) that inherits from the base `RasterizerClass`. Acts as a thin wrapper layer for software-based polygon and rectangle rasterization, delegating actual rendering logic to the "scottish_textures.c" module.

## Core Responsibilities
- Provide a concrete software rasterizer implementation for the engine's rendering pipeline
- Maintain pointers to view data and screen bitmap used by the texture rasterization system
- Declare the interface for rendering textured horizontal polygons, vertical polygons, and rectangles
- Serve as the abstraction layer between the game engine's renderer and low-level software rasterization code

## External Dependencies
- **Includes**: `Rasterizer.h` (base class definition)
- **Defined elsewhere**: 
  - `view_data` (render.h or included via Rasterizer.h)
  - `bitmap_definition` (render.h or included via Rasterizer.h)
  - `polygon_definition` (render.h or included via Rasterizer.h)
  - `rectangle_definition` (render.h or included via Rasterizer.h)
  - `RasterizerClass` (Rasterizer.h)
  - Implementation of texture rendering methods (scottish_textures.c)

# Source_Files/RenderMain/render.cpp
## File Purpose
Core rendering pipeline orchestrator for the Aleph One game engine. Manages view initialization, coordinates the multi-stage rendering process (visibility, sorting, object placement, rasterization), handles camera effects, and renders the player's HUD/weapons layer. Serves as the main entry point between the game world and graphics output.

## Core Responsibilities
- Memory allocation and initialization for rendering system (`allocate_render_memory`)
- View/camera parameter setup and per-frame updates (`initialize_view_data`, `update_view_data`)
- Main rendering loop orchestration (`render_view`), dispatching to specialized subsystems
- Render effect application (earthquakes, teleport folds) with phase tracking
- Transfer mode instantiation for polygons and sprites (texture animations, fades, static, etc.)
- First-person weapon/HUD rendering (`render_viewer_sprite_layer`)
- Sprite positioning and viewport calculations

## External Dependencies
- **Notable includes:** map.h (geometry), render.h (declarations), interface.h (shapes/UI), lightsource.h, media.h, weapons.h, RenderVisTree.h, RenderSortPoly.h, RenderPlaceObjs.h, RenderRasterize.h (subsystem classes), Rasterizer_SW.h, Rasterizer_OGL.h, AnimatedTextures.h, OGL_Render.h.
- **Key external symbols:** `get_polygon_data()`, `get_endpoint_data()`, `get_light_intensity()`, `get_media_data()`, `get_weapon_display_information()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `OGL_IsActive()`, `OGL_GetModelData()`, `render_overhead_map()`, `render_computer_interface()` (defined elsewhere).
- **Trigonometry tables:** `sine_table[]`, `cosine_table[]` from world.h.

# Source_Files/RenderMain/render.h
## File Purpose

Header for the core rendering system of the Aleph One game engine (Marathon-compatible). Defines the main view/camera data structure, rendering state management, visibility flags for BSP culling, and prototypes for frame rendering, effects, and UI overlays.

## Core Responsibilities

- Define `view_data` struct containing all camera/viewport state (position, orientation, FOV, screen dimensions, projection parameters)
- Provide visibility culling flags and macros for portal/BSP traversal (polygon/endpoint/side visibility bits)
- Declare main render entry point (`render_view`) and frame-setup functions
- Manage rendering effects (fold-in/out, explosions, tunnel vision state)
- Provide overhead map and computer terminal UI rendering
- Support transfer mode instantiation for textured surfaces (tinting, static, landscape effects)
- Track view modes (terminal, overhead map, tunnel vision, media boundary)

## External Dependencies

- **world.h** ΓÇö `world_point3d`, `world_vector2d/3d`, `world_distance`, angle types, trig tables, coordinate transforms
- **textures.h** ΓÇö `bitmap_definition` struct for surface textures
- **ViewControl.h** ΓÇö Field-of-view accessors (`View_FOV_Normal()`, etc.), landscape and effect control flags
- **scottish_textures.h** ΓÇö `rectangle_definition`, `polygon_definition`, transfer mode constants, shape descriptors

# Source_Files/RenderMain/RenderPlaceObjs.cpp
## File Purpose
Manages placement of in-game objects (sprites, 3D models) into the sorted polygon rendering order for the Aleph One game engine. Handles 2D projection, depth sorting, clipping window calculation, and parasitic object linking to produce a renderable object list in depth order.

## Core Responsibilities
- Initialize and manage render object lists
- Transform world-space objects to screen space with 2D/3D projection
- Sort render objects into the rendering tree based on depth and polygon crossing
- Calculate aggregate clipping windows for objects spanning multiple polygons
- Project 3D model bounding boxes to sprite rectangles for proper depth sorting
- Handle parasitic objects (objects attached to host objects)
- Support object scaling (enlarged/tiny) and transfer modes (fade/fold/etc.)
- Integrate with OpenGL 3D model rendering and chase-cam opacity

## External Dependencies
- **Includes:** `map.h`, `lightsource.h`, `media.h`, `OGL_Setup.h`, `ChaseCam.h`, `player.h`, `RenderPlaceObjs.h`
- **Defined elsewhere:** `get_polygon_data()`, `get_light_intensity()`, `get_object_data()`, `get_endpoint_data()`, `get_object_shape_and_transfer_mode()`, `extended_get_shape_information()`, `extended_get_shape_bitmap_and_shading_table()`, `rescale_shape_information()`, `instantiate_rectangle_transfer_mode()`, `OGL_GetModelData()` (conditional HAVE_OPENGL), `get_dynamic_limit()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `current_player`, `GetChaseCamData()`, trig tables (cosine_table, sine_table), `isqrt()`, `PIN()`, `normalize_angle()`
- **External types:** `view_data`, `sorted_node_data`, `polygon_data`, `object_data`, `endpoint_data`, `shape_information_data`, `clipping_window_data`, `rectangle_definition`, `OGL_ModelData`, `RenderVisTreeClass`, `RenderSortPolyClass`

# Source_Files/RenderMain/RenderPlaceObjs.h
## File Purpose
Defines the `RenderPlaceObjsClass` for spatial organization and depth-sorting of game objects (inhabitants) within the rendering pipeline. Objects are placed into a tree structure aligned with visible polygons and culled by visibility windows. Created by Loren Petrich; migrated from monolithic render.c to use STL vectors instead of custom growable lists.

## Core Responsibilities
- Build and populate a list of renderable objects with world positions and light intensities
- Construct render objects with calculated clipping windows for visibility culling
- Sort objects into a spatial tree keyed by polygon/node for depth ordering
- Calculate which polygons ("base nodes") are relevant to an object's position
- Rescale shape geometry based on distance and screen projection
- Integrate with the visibility tree (RenderVisTree) and polygon sorter (RenderSortPoly)

## External Dependencies
- **STL:** `<vector>` 
- **Engine core:** `world.h` (3D coordinates, `long_point3d`, `world_point3d`, `world_distance`), `interface.h` (`shape_information_data`), `render.h` (`view_data`, `_fixed`), `RenderSortPoly.h` (`RenderSortPolyClass`, `sorted_node_data`, `clipping_window_data`)
- **External symbols** (not defined here): `view_data`, `sorted_node_data`, `clipping_window_data`, `RenderVisTreeClass`, `RenderSortPolyClass`, shape scaling/transformation logic

# Source_Files/RenderMain/RenderRasterize.cpp
## File Purpose

Implements polygon clipping, coordinate transformation, and rasterization calls for a 3D game engine. Converts world-space polygon data into screen-space geometry, clips to view windows, and delegates textured rendering to a rasterizer backend. Handles floors, ceilings, walls, liquids, and sprite objects with proper depth ordering and lighting.

## Core Responsibilities

- Traverse sorted BSP/polygon tree and render visible geometry back-to-front
- Clip horizontal polygons (floors/ceilings) and vertical polygons (walls) to camera frustum
- Transform clipped vertices from world-space to perspective-correct screen-space
- Handle semi-transparent liquid surfaces with optional see-through rendering  
- Calculate and apply surface texturing, lighting, and transfer modes
- Clip sprite objects relative to liquid boundaries and view windows
- Support both 2D and 3D clipping algorithms (XY, Z, XZ planes)
- Translate animated textures and manage void-presence flags for transparency

## External Dependencies

- **map.h**: `polygon_data`, `endpoint_data`, `line_data`, `side_data`, `get_*_data()` accessors
- **lightsource.h**: `get_light_intensity()`
- **media.h**: `media_data`, `get_media_data()`
- **RenderRasterize.h**: Class definition, forward decls
- **AnimatedTextures.h**: `AnimTxtr_Translate()`
- **OGL_Setup.h**: `Get_OGL_ConfigureData()`, `TEST_FLAG()` macro
- **preferences.h**: `graphics_preferences` (alpha blending mode)
- **screen.h**: `get_screen_mode()`
- **csmacros.h** (via cseries.h): `TEST_RENDER_FLAG()`, `WRAP_HIGH()`, `WRAP_LOW()`, `SGN()`, `PIN()`, `FIXED_*` macros, `WORLD_ONE`
- **Rasterizer.h**: `RasterizerClass` with `texture_horizontal_polygon()`, `texture_vertical_polygon()`, `texture_rectangle()`
- **RenderSortPoly.h**: `RenderSortPolyClass` with `SortedNodes` vector
- **string.h**: `memmove()`

Defined elsewhere:
- `overflow_short_to_long_2d()`, `get_shape_bitmap_and_shading_table()`, `instantiate_polygon_transfer_mode()` (render utilities)
- `TEST_RENDER_FLAG()` (side visibility check)
- `CROSSPROD_TYPE`, `FIXED_FRACTIONAL_BITS`, `WORLD_FRACTIONAL_PART()` (fixed-point/coordinate types)

# Source_Files/RenderMain/RenderRasterize.h
## File Purpose
Defines the `RenderRasterizerClass`, which rasterizes (draws) visible world geometry to the screen. Performs coordinate transformation, viewport clipping, and delegates final rendering to a backend `RasterizerClass`. Part of the Aleph One game engine's rendering pipeline (after visibility tree construction and polygon depth-sorting).

## Core Responsibilities
- Render visible floors, ceilings, walls, and game objects to the framebuffer
- Clip world-space geometry to the viewport frustum (XY and Z bounds)
- Handle long-distance rendering by tracking coordinate overflow in flags
- Distinguish rendering for "void" vs. filled surfaces and media boundaries
- Coordinate between the visibility tree, sorted polygons, and object placement layers
- Transform and validate polygon vertex counts before rasterization

## External Dependencies
- **world.h** ΓÇö coordinate types: `world_distance`, `long_vector2d`, `world_point3d`, `int32`
- **render.h** ΓÇö `view_data`, `polygon_data`, `clipping_window_data`
- **RenderSortPoly.h** ΓÇö `RenderSortPolyClass` (sorted polygons)
- **RenderPlaceObjs.h** ΓÇö `render_object_data` (game objects)
- **Rasterizer.h** ΓÇö `RasterizerClass` (abstract backend)
- **\<vector\>** ΓÇö STL for dynamic arrays
- **Defined elsewhere:** `render_object_data`, `polygon_data`, `horizontal_surface_data`, `clipping_window_data`

# Source_Files/RenderMain/RenderSortPoly.cpp
## File Purpose
Implements polygon depth-sorting for the render pipeline. Decomposes the render visibility tree into a depth-ordered list of sorted nodes, computing clipping windows for each polygon to constrain rendering boundaries.

## Core Responsibilities
- Sorts polygons into back-to-front rendering order via tree decomposition
- Builds clipping windows that define screen-space bounds for polygon rendering
- Accumulates clipping constraints from polygon parents and ancestors
- Calculates vertical clip data (top/bottom) covering specified horizontal runs
- Manages resizable STL vector containers for sorted nodes and clipping data

## External Dependencies
- **Includes:** `cseries.h` (macros, utilities), `map.h` (polygon_data, world structures), `RenderSortPoly.h` (class definition), `<string.h>`, `<limits.h>` (SHRT_MIN/MAX)
- **External symbols:** `get_polygon_data(short)`, `TEST_RENDER_FLAG()`, `TEST_FLAG()`, `csprintf()`, `vassert()`, `POINTER_CAST()`, `POINTER_DATA` (pointer arithmetic macros); `node_data`, `clipping_window_data`, `endpoint_clip_data`, `line_clip_data` (defined in render.h); `RenderVisTreeClass` (defined in RenderVisTree.h); STL `std::vector` template

# Source_Files/RenderMain/RenderSortPoly.h
## File Purpose
Defines a C++ class for sorting world polygons into appropriate depth order for rendering. Integrates with the visibility tree to organize sorted rendering nodes, manage clipping windows, and calculate vertex clipping data for the rasterization stage.

## Core Responsibilities
- Sort polygons into depth-order using visibility tree data
- Initialize and maintain the sorted render node tree structure
- Build and manage clipping windows for polygon culling
- Calculate vertical clipping bounds for rendering edges
- Accumulate and resize internal clip data structures
- Provide mapping from map polygon indices to sorted nodes

## External Dependencies
- **Includes:** `<vector>` (STL), `world.h` (world coordinate types), `render.h` (view/render flags), `RenderVisTree.h` (visibility tree definition)
- **Defined elsewhere:** `view_data` (render.h), `node_data` (RenderVisTree.h), `render_object_data` (not shown), `clipping_window_data` (RenderVisTree.h), `line_clip_data` / `endpoint_clip_data` (RenderVisTree.h)

# Source_Files/RenderMain/RenderVisTree.cpp
## File Purpose
Implements a visibility-tree builder for the 3D rendering pipeline. The class traverses map geometry via ray-casting from a viewpoint, constructing a hierarchical tree of visible polygons and calculating clipping planes for efficient rendering of geometry and effects.

## Core Responsibilities
- Building a polygon visibility tree via recursive ray-casting from the camera viewpoint
- Queueing and processing polygons in breadth-first order
- Calculating screen-space clipping information (lines and endpoints that constrain rendering)
- Managing endpoint transformations and long-distance overflow handling
- Constructing a polygon-sorted tree (binary search structure) for efficient polygon lookups
- Detecting polygon transitions and elevation changes across transparent map boundaries
- Handling ray splits when rays pass through vertices exactly

## External Dependencies
- **cseries.h** ΓÇö Common series utilities (macros, types, assertions).
- **map.h** ΓÇö World geometry definitions (`polygon_data`, `endpoint_data`, `line_data`, `world_point2d`, world distance types, accessor functions).
- **render.h** ΓÇö Rendering declarations (included via RenderVisTree.h; defines `view_data`).
- **RenderVisTree.h** ΓÇö Class definition and helper macros (`WRAP_LOW`, `WRAP_HIGH`, `POINTER_CAST`, `CROSSPROD_TYPE`).
- **Functions defined elsewhere:** `get_polygon_data()`, `get_endpoint_data()`, `get_line_data()`, `transform_overflow_point2d()`, `overflow_short_to_long_2d()`, `short_to_long_2d()`, `PIN()`, `SWAP()` (macro or inline).
- **Global flags/macros:** `TEST_RENDER_FLAG()`, `SET_RENDER_FLAG()`, `_polygon_is_visible`, `_endpoint_has_been_visited`, `_endpoint_has_been_transformed`, `_endpoint_has_clip_data`, `_line_has_clip_data`, `ENDPOINT_IS_TRANSPARENT()`, `LINE_IS_TRANSPARENT()`, `ADD_POLYGON_TO_AUTOMAP()`, `ADD_LINE_TO_AUTOMAP()`, `SET_RENDER_FLAG()`.
- **Constants:** `NONE` (likely -1 or sentinel), polygon/endpoint/line index enums.

# Source_Files/RenderMain/RenderVisTree.h
## File Purpose
Defines `RenderVisTreeClass`, a visibility tree structure extracted from `render.c` for determining polygon visibility from the viewer's perspective. Manages the tree traversal, polygon queueing, and screen-space clipping calculations needed for efficient rendering.

## Core Responsibilities
- Build and traverse a visibility tree rooted at the viewpoint polygon
- Track which polygons and sides are visible within the view cone
- Manage screen-boundary clipping data for vertices and line segments
- Calculate screen-space coordinates for visible endpoints
- Support ray-casting and polygon adjacency traversal
- Maintain polygon sort tree for depth ordering across visibility tree

## External Dependencies
- **Includes:** `<vector>` (STL), `world.h`, `render.h`
- **Defined elsewhere:**
  - `view_data` (render.h) ΓÇö view state with FOV, origin, pitch/yaw
  - `world_point2d`, `long_vector2d` (world.h) ΓÇö geometry primitives
  - Map polygon/endpoint/line data (via world.h)
  - `MAXIMUM_VERTICES_PER_POLYGON` constant (render.h)

# Source_Files/RenderMain/scottish_textures.cpp
## File Purpose
Software texture rasterizer for the Marathon/Aleph One game engine. Maps 2D textures onto screen-space polygons and rectangles using precalculated coordinate tables and Bresenham-style line algorithms, with support for multiple color depths, alpha blending, and landscape textures.

## Core Responsibilities
- Texture-map convex polygons (both horizontal and vertical scan-line orientations)
- Texture-map axis-aligned rectangles with perspective correction
- Landscape/terrain texture rendering with tiling and repeat options
- Precalculate texture coordinates and shading tables before rasterization
- Build Bresenham line-drawing tables for polygon edge traversal
- Dispatch to templated low-level pixel writers (8/16/32-bit, with/without alpha)
- Handle multiple transfer modes (textured, static/randomized, tinted, landscaped)

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö Core types, assertions, utilities
  - `render.h` ΓÇö `polygon_definition`, `rectangle_definition`, `view_data`, `bitmap_definition`, render flags
  - `Rasterizer_SW.h` ΓÇö `Rasterizer_SW_Class` method container
  - `preferences.h` ΓÇö `graphics_preferences` global (for alpha blending options)
  - `SW_Texture_Extras.h` ΓÇö `SW_Texture_Extras` singleton for per-shape alpha tables
  - `low_level_textures.h` ΓÇö Templated pixel writing functions (`texture_horizontal_polygon_lines<>()`, `texture_vertical_polygon_lines<>()`, `landscape_horizontal_polygon_lines<>()`, `randomize_vertical_polygon_lines<>()`, `tint_vertical_polygon_lines<>()`)

- **External globals used:**
  - `bit_depth` ΓÇö Current pixel color depth (8, 16, or 32 bits)
  - `graphics_preferences` ΓÇö Rendering preferences (software alpha blending mode)
  - `number_of_shading_tables` ΓÇö Count of lighting lookup tables
  - `cosine_table[]`, `sine_table[]` ΓÇö Trigonometric lookup tables
  - `FIXED_FRACTIONAL_BITS`, `WORLD_FRACTIONAL_BITS`, `ANGULAR_BITS`, `TRIG_SHIFT` ΓÇö Fixed-point precision constants

- **External functions called:**
  - `View_GetLandscapeOptions(shape_descriptor)` ΓÇö Retrieve landscape tiling parameters (defined elsewhere)
  - `SW_Texture_Extras::instance()->GetTexture(shape_descriptor)` ΓÇö Retrieve per-shape opacity table (if alpha blending enabled)

# Source_Files/RenderMain/scottish_textures.h
## File Purpose
Defines core texture and surface rendering data structures for the Aleph One game engine (Marathon-based). Establishes contracts for rendering 2D rectangles (sprites/objects) and 3D polygons (walls/surfaces) with support for multiple transfer modes, shading tables, and both software and OpenGL rendering paths.

## Core Responsibilities
- Define transfer modes (tinting, solid, textured, shadeless, static) for texture rendering
- Define tint table structures for 8, 16, and 32-bit color depths
- Declare `rectangle_definition` struct for sprite/object rendering (weapons, items, scenery)
- Declare `polygon_definition` struct for wall/surface polygon rendering
- Expose shading table management state and initialization entry point
- Support both 2D (software rasterizer) and 3D (OpenGL model) rendering paths

## External Dependencies
- **Includes:** `shape_descriptors.h` (defines `shape_descriptor` typedef; collection enums)
- **Forward declared:** `OGL_ModelData` (OpenGL 3D model container, defined elsewhere)
- **Undefined types used:** `pixel8`, `pixel16`, `pixel32` (color formats, likely from pixel-format header); `_fixed` (fixed-point); `world_point3d`, `world_vector3d`, `point2d` (3D math); `GLfloat` (OpenGL)
- **Referenced elsewhere:** Rendering code, shape/collection management, OpenGL subsystem

# Source_Files/RenderMain/shape_definitions.h
## File Purpose
Defines the data structure for collection headers that manage shape/sprite asset collections in the rendering system. Acts as the primary interface between the asset loader and the rendering pipeline, storing metadata and pointers to collection definitions and shading tables.

## Core Responsibilities
- Define the `collection_header` struct (32 bytes) to store collection metadata on disk
- Track offsets and lengths for both standard and 16-bit format collection data
- Maintain pointers to runtime collection definitions and shading lookup tables
- Provide the global array of collection headers indexed by collection ID

## External Dependencies
- **Defined elsewhere**: `collection_definition` (referenced via pointer), `byte` and integer type aliases (`int16`, `uint16`, `int32`)
- **Assumed external**: `MAXIMUM_COLLECTIONS` constant (likely defined in a config or constants header)

---

**Notes:**
- The structure is exactly 32 bytes as documented, matching a likely fixed-size disk format
- The comment from Aug 2000 indicates a refactoring from MacOS-specific handles to pointers for cross-platform compatibility
- Status and flags fields suggest collection lifecycle management (loaded, active, etc.)

# Source_Files/RenderMain/shape_descriptors.h
## File Purpose
Defines the binary layout and macros for 16-bit shape descriptors used to index 3D graphics shapes in Aleph One (Marathon engine port). Encodes collection, shape index, and color lookup table (CLUT) information into a single uint16 value.

## Core Responsibilities
- Define `shape_descriptor` type and bit-field layout (8 bits shape, 5 bits collection, 3 bits CLUT)
- Enumerate all 32 game object/scenery collections (weapons, enemies, walls, landscape, etc.)
- Provide macros to extract shape and collection indices from descriptors
- Provide macros to construct descriptors and collection values with CLUTs
- Define maximum limits for each descriptor component

## External Dependencies
None. Self-contained definition header; used throughout render pipeline and sprite systems.

# Source_Files/RenderMain/shapes.cpp
## File Purpose
Manages game shape (sprite/texture) collections including loading, rendering, and visual effects. Handles shape data parsing, bitmap unpacking, shading table generation, and SDL surface creation for 2D rendering with support for RLE-encoded shapes and dynamic illumination.

## Core Responsibilities
- Load and cache shape collections from binary resource files
- Unpack RLE-encoded bitmap data and create SDL surfaces
- Build color shading tables for darkness/lighting effects and infravision
- Manage collection lifecycle (load, strip, unload) with memory pooling
- Apply per-pixel illumination, mirroring, and shrinking transformations
- Parse and apply infravision tint configuration via XML
- Provide accessors for high-level, low-level, and bitmap shape definitions

## External Dependencies
- **SDL (audio/graphics):** SDL_RWops, SDL_ReadBE16/32, SDL_MapRGB, SDL_CreateRGBSurfaceFrom, SDL_SetColors, SDL_SetColorKey, SDL_GetVideoSurface, SDL_PixelFormat, SDL_SwapBE16, SDL_CreateRGBSurfaceFrom.
- **Collection metadata:** collection_definition.h defines struct layouts; collection_header is declared in shape_definitions.h (not shown).
- **File I/O:** FileHandler.h (OpenedFile class); shapes file opened via open_shapes_file() (defined elsewhere).
- **OpenGL:** OGL_Render.h, OGL_LoadScreen.h; OGL_SetInfravisionTint() called conditionally.
- **Color parsing:** ColorParser.h; Color_GetParser() and Color_SetArray() used for XML.
- **Byte order:** byte_swapping.h; byte_swap_memory() for endian conversion.
- **Texture extras:** SW_Texture_Extras.h, Packing.h (headers included; purpose unclear from this file).
- **XML parsing:** XML_ElementParser base class; cseries.h and interface.h for macros and assertions.
- **Profiling:** psp_sdl_profilermg.h for PSP target (PSPROF_* macros).

# Source_Files/RenderMain/SW_Texture_Extras.cpp
## File Purpose

Manages software-rendered texture opacity tables and their XML configuration. Builds per-texture opacity lookup tables based on shading data and bit depth, and provides XML parsing for texture opacity settings (type, scale, shift). Part of the software rendering pipeline for texture management.

## Core Responsibilities

- Build opacity lookup tables from shading tables for 16-bit and 32-bit color modes
- Manage texture storage across multiple collections (2D array indexed by collection and bitmap)
- Load/unload opacity tables for entire collections
- Parse XML texture definitions and apply opacity parameters
- Singleton management of global texture extras state

## External Dependencies

- **SDL:** `SDL_GetVideoSurface()`, `SDL_PixelFormat` (pixel format query)
- **Shapes system:** `get_shape_bitmap_and_shading_table()`, `bitmap_definition`, shape descriptor macros (`GET_COLLECTION`, `GET_DESCRIPTOR_*`, `BUILD_DESCRIPTOR`)
- **XML parsing:** `XML_ElementParser` base class, `ReadBoundedInt16Value()`, `ReadFloatValue()`, `StringsEqual()`, `UnrecognizedTag()`
- **Utility:** `PIN()` macro (clamp to range)
- **Collection/shape definitions:** `collection_definition.h`, `shape_descriptors.h`, `MAXIMUM_SHADING_TABLE_INDEXES`, `NUMBER_OF_COLLECTIONS`, `MAXIMUM_SHAPES_PER_COLLECTION`

# Source_Files/RenderMain/SW_Texture_Extras.h
## File Purpose
Defines classes for managing software (non-hardware) textures with opacity/transparency tables in the Aleph One game engine. Provides a singleton interface to store, retrieve, and configure per-shape texture metadata across all shape collections.

## Core Responsibilities
- Store texture metadata (shape descriptor, opacity type, scale/shift parameters) for individual shapes
- Maintain opacity lookup tables (8-bit alpha values) for texture transparency
- Implement a singleton manager to organize textures by collection ID
- Support per-collection load/unload lifecycle for texture resources
- Integrate with XML-based configuration parsing

## External Dependencies
- `cseries.h`, `cstypes.h`: Core types (`uint8`, etc.)
- `shape_descriptors.h`: `shape_descriptor` type; `NUMBER_OF_COLLECTIONS` constant
- `XML_ElementParser.h`: Base class for XML parsing
- `<vector>`: STL vector storage

# Source_Files/RenderMain/textures.cpp
## File Purpose
Bitmap and texture memory management utilities for the Aleph One game engine. Handles bitmap origin calculation, row address precalculation for both linear and RLE-compressed formats, and palette/color remapping operations.

## Core Responsibilities
- Calculate the memory origin (starting address) of bitmap pixel data within a bitmap_definition structure
- Precalculate row address lookup tables for efficient row access in both row-order and column-order layouts
- Support both linear (standard) and RLE-compressed (transparent shape) bitmap formats
- Handle Marathon 1 and Marathon 2 RLE format variants with different endianness conventions
- Apply color/palette remapping to entire bitmaps via byte-by-byte lookup table substitution

## External Dependencies
- **Notable includes:** `cseries.h` (base types, macros), `textures.h` (bitmap_definition)
- **Conditional compilation:** `#ifdef MARATHON1` / `#ifdef MARATHON2` for format-specific RLE handling
- **External symbols used:** `bitmap_definition`, `pixel8`, `byte`, `NONE` macro (likely -1), `_COLUMN_ORDER_BIT` flag

# Source_Files/RenderMain/textures.h
## File Purpose
Header file defining bitmap texture structures and utility functions for the Aleph One game engine. Provides memory layout definitions and pixel data manipulation routines for in-game textures.

## Core Responsibilities
- Define bitmap texture data structure (`bitmap_definition`) with metadata and row addressing
- Calculate bitmap memory origin and row address tables
- Support bitmap color remapping via palette lookup tables
- Track bitmap flags (column order, transparency, patching status)

## External Dependencies
- `cseries.h` ΓÇö provides `pixel8`, `int16`, `byte` type definitions and cross-platform SDL setup
- Assumes pixel data layout is immediately after struct in memory (no indirection)


