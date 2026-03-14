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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `OGL_TextureOptionsBase` | struct | Container for texture parameters: opacity settings, file paths, image descriptors, blend modes, and image dimensions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ALL_CLUTS` | const int | global | Sentinel value (ΓêÆ1) indicating all color tables apply |
| `OGL_OpacType_*` | enum constants | global | Opacity type identifiers (Crisp, Flat, Avg, Max) |
| `OGL_BlendType_*` | enum constants | global | Blend mode identifiers (Crossfade, Add, with Premult variants) |
| `INFRAVISION_BITMAP_SET` | enum constant | global | Bitmap set index for infravision images |
| `SILHOUETTE_BITMAP_SET` | enum constant | global | Bitmap set index for silhouette images |

## Key Functions / Methods

### OGL_TextureOptionsBase::Load
- **Signature:** `void Load();`
- **Purpose:** Load texture image data from configured file paths (NormalColors, NormalMask, GlowColors, GlowMask)
- **Inputs:** File paths in `vector<char>` members; opacity/blend settings; dimension hints
- **Outputs/Return:** Populates `NormalImg` and `GlowImg` ImageDescriptors
- **Side effects:** Allocates GPU/CPU memory for pixel data; reads files from disk
- **Calls:** Not visible; implemented elsewhere (likely ImageLoader)

### OGL_TextureOptionsBase::Unload
- **Signature:** `void Unload();`
- **Purpose:** Release texture image memory and clear loaded image data
- **Inputs:** None (operates on object state)
- **Outputs/Return:** None
- **Side effects:** Deallocates pixel buffers; clears `NormalImg` and `GlowImg`
- **Calls:** Not visible; likely delegates to ImageDescriptor cleanup

### OGL_TextureOptionsBase::GetMaxSize
- **Signature:** `virtual int GetMaxSize();`
- **Purpose:** Query maximum texture dimension supported (likely GPU-dependent)
- **Inputs:** None
- **Outputs/Return:** Maximum size in pixels
- **Side effects:** None
- **Calls:** Not visible
- **Notes:** Virtual; overridden in derived classes for different texture types (wall vs. sprite vs. skin)

## Control Flow Notes

This is a definition/configuration file with no control flow. Used during engine initialization (texture resource loading phase) and render setup (blending/opacity application). Constructor `OGL_TextureOptionsBase()` initializes all members with safe defaults (Crisp opacity, Crossfade blend, no substitution).

## External Dependencies

- **Standard Library:** `<vector>` (std::vector for file paths)
- **Internal Headers:**
  - `shape_descriptors.h` ΓÇö defines `shape_descriptor`, collection/CLUT constants (`MAXIMUM_CLUTS_PER_COLLECTION`)
  - `ImageLoader.h` ΓÇö defines `ImageDescriptor` class for holding loaded pixel data
- **Conditional:** Entire file guarded by `#ifdef HAVE_OPENGL`
