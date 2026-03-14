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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Color8888` | struct | 32-bit RGBA color with byte-level channel access |
| `Color565` | struct | 16-bit RGB color with 5-6-5 bit packing (DXTC source format) |
| `DXTColBlock` | struct | DXTC1/3/5 color block: two color endpoints + 4├ù4 pixel indices |
| `DXTAlphaBlockExplicit` | struct | DXTC3 alpha block: explicit 4-bit alpha per pixel |
| `DXTAlphaBlock3BitLinear` | struct | DXTC5 alpha block: two alpha endpoints + 3-bit interpolated indices |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `log2` | function | static inline | Compute base-2 logarithm (non-PSP platforms only) |
| `padfour` | function | static inline | Pad value to nearest multiple of 4 |

## Key Functions / Methods

### ImageDescriptor::GetMipMapSize
- Signature: `int GetMipMapSize(int level) const`
- Purpose: Calculate memory size of a mipmap level given the base image dimensions and format.
- Inputs: `level` ΓÇô mipmap index (0 = full size, each increment halves dimensions)
- Outputs/Return: Size in bytes; 0 if level is invalid.
- Side effects: Prints to stderr on invalid level.
- Calls: (none visible)
- Notes: Accounts for DXTC block alignment (4├ù4 pixel blocks); formula differs per format (RGBA8 = W├ùH├ù4, DXTC1 = (W/4)├ù(H/4)├ù8, DXTC3/5 = (W/4)├ù(H/4)├ù16).

### ImageDescriptor::GetMipMapPtr (both const and non-const)
- Signature: `const uint32 *GetMipMapPtr(int Level) const` / `uint32 *GetMipMapPtr(int Level)`
- Purpose: Get a pointer into the pixel buffer to the start of a given mipmap level.
- Inputs: `Level` ΓÇô mipmap index
- Outputs/Return: Pointer to mipmap data; NULL if offset exceeds buffer size.
- Side effects: (none)
- Calls: `GetMipMapSize`, `GetBuffer`, `GetBufferSize`
- Notes: Accumulates sizes of all prior levels; pointer arithmetic assumes uint32 alignment.

### ImageDescriptor::Resize (both overloads)
- Signature: `void Resize(int _Width, int _Height)` / `void Resize(int _Width, int _Height, int _TotalBytes)`
- Purpose: Reallocate the pixel buffer to new dimensions (first form infers size; second form explicit).
- Inputs: New width, height; optionally total byte count.
- Outputs/Return: (void)
- Side effects: Deletes old `Pixels` array and allocates new one.
- Calls: `delete []`, `new`
- Notes: First form assumes RGBA8 (4 bytes/pixel); no validation that new size matches format.

### ImageDescriptor::Minify
- Signature: `bool Minify()`
- Purpose: Reduce image dimensions by half (used for downsampling or removing the highest mipmap level).
- Inputs: (none)
- Outputs/Return: `true` if successful; `false` if image cannot be further minified or GL unavailable.
- Side effects: Reallocates pixel buffer, updates `Width`, `Height`, `Size`, `MipMapCount`.
- Calls: `GetMipMapSize`, `GetMipMapPtr`, `delete []`, `new`, `OGL_IsActive`, `gluScaleImage` (OpenGL)
- Notes: If mipmaps exist, removes the highest-resolution level. Otherwise uses OpenGL to downscale RGBA8 (fails if OpenGL inactive). Prints to stderr on GL failure.

### ImageDescriptor::LoadDDSFromFile
- Signature: `bool LoadDDSFromFile(FileSpecifier& File, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- Purpose: Main DDS file loader; parses header, validates format, and loads texture data (with optional mipmap handling and power-of-two resizing).
- Inputs: 
  - `File` ΓÇô file specifier pointing to DDS file
  - `flags` ΓÇô bitmask (ImageLoader_ResizeToPowersOfTwo, ImageLoader_CanUseDXTC, ImageLoader_LoadMipMaps, ImageLoader_LoadDXTC1AsDXTC3)
  - `actual_width`, `actual_height` ΓÇô override DDS dimensions for UV scaling calculation
  - `maxSize` ΓÇô maximum texture dimension; larger textures are downsampled
- Outputs/Return: `true` on success; `false` on any validation/read error.
- Side effects: Opens file, reallocates pixel buffer, updates `Width`, `Height`, `Format`, `MipMapCount`, `UScale`, `VScale`.
- Calls: `File.Open`, `dds_file.Read`, `AIStreamLE` stream parsing, `NextPowerOfTwo`, `SkipMipMapFromFile`, `LoadMipMapFromFile`, `Minify`, `MakeRGBA`, `MakeDXTC3`
- Notes: Validates DDS magic ("DDS "), rejects cubemaps and volume textures. Handles mipmap chains; recomputes mipmap count if resized to power-of-two. Supports format detection (RGB vs DXTC FourCC). Prints errors to stderr.

### ImageDescriptor::LoadMipMapFromFile
- Signature: `bool LoadMipMapFromFile(OpenedFile& file, int flags, int level, DDSURFACEDESC2 &ddsd, int skip)`
- Purpose: Read and decode a single mipmap level from file into the pixel buffer at the appropriate offset.
- Inputs:
  - `file` ΓÇô open DDS file
  - `flags` ΓÇô loading flags
  - `level` ΓÇô mipmap level to read
  - `ddsd` ΓÇô DDS surface descriptor
  - `skip` ΓÇô number of levels already skipped (for offset calculation)
- Outputs/Return: `true` on successful read; `false` on buffer overflow or file read error.
- Side effects: Reads from file, writes to pixel buffer.
- Calls: `GetMipMapSize`, `GetBuffer`, `file.Read`, `SDL_CreateRGBSurfaceFrom`, `SDL_SetAlpha`, `SDL_BlitSurface`, `SDL_FreeSurface`, `memset`, `padfour`
- Notes: For RGBA8, uses SDL surface blitting to handle format conversion and pitch correction. For DXTC formats, directly copies compressed blocks (with 4├ù4 block alignment padding). Endianness-aware (ALEPHONE_LITTLE_ENDIAN).

### ImageDescriptor::SkipMipMapFromFile
- Signature: `bool SkipMipMapFromFile(OpenedFile& File, int flags, int level, DDSURFACEDESC2 &ddsd)`
- Purpose: Advance file position past a mipmap level without reading it into memory.
- Inputs: Level descriptor and DDS surface info.
- Outputs/Return: `true` if seek succeeded; `false` on file error.
- Side effects: Modifies file position.
- Calls: `File.GetPosition`, `File.SetPosition`, `padfour`
- Notes: Format-aware; calculates skip distance differently for RGBA8 vs DXTC.

### ImageDescriptor::MakeRGBA
- Signature: `bool MakeRGBA()`
- Purpose: Convert image from DXTC format to RGBA8 (decompression).
- Inputs: (none; operates on current image state)
- Outputs/Return: `true` on success; `false` if format is not DXTC or decompression fails.
- Side effects: Reallocates pixel buffer, updates `Format`, `Size`, `Pixels`.
- Calls: `DecompressDXTC1`, `DecompressDXTC3`, `DecompressDXTC5`, `new`, `delete []`, `GetMipMapPtr`, `GetMipMapSize`
- Notes: Processes all mipmap levels independently.

### ImageDescriptor::MakeDXTC3
- Signature: `bool MakeDXTC3()`
- Purpose: Convert DXTC1 texture to DXTC3 by inserting opaque alpha blocks.
- Inputs: (none)
- Outputs/Return: `true` on success; `false` if format is not DXTC1.
- Side effects: Doubles buffer size, updates `Format` and `Size`.
- Calls: `new`, `memset`, `memcpy`, `delete []`
- Notes: Creates explicit alpha blocks (0xff = fully opaque) for each DXTC1 block. Useful for ensuring consistent alpha channel presence.

### ImageDescriptor::PremultiplyAlpha
- Signature: `void PremultiplyAlpha()`
- Purpose: Premultiply RGB channels by alpha to enable correct blending (standard technique for premultiplied-alpha compositing).
- Inputs: (none)
- Outputs/Return: (void)
- Side effects: Modifies pixel data in-place, sets `PremultipliedAlpha = true`.
- Calls: (none)
- Notes: Skips fully opaque (alpha=255) and fully transparent (alpha=0) pixels as optimizations. Uses integer arithmetic with rounding (add 127 before divide by 255). Endianness-aware byte access.

### DecompressDXTC1, DecompressDXTC3, DecompressDXTC5
- Signature: `static bool DecompressDXTC*(uint32 *out, int width, int height, uint32 *in)`
- Purpose: Decompress DXTC block-compressed data into RGBA8 uncompressed output.
- Inputs: 
  - `out` ΓÇô destination RGBA8 buffer (uint32 array, treated as byte array for endianness)
  - `width`, `height` ΓÇô texture dimensions
  - `in` ΓÇô source DXTC compressed data
- Outputs/Return: `true` (always; no error detection)
- Side effects: Writes decompressed pixels to `out`.
- Calls: `SDL_SwapLE16`, `SDL_SwapLE32`, (color interpolation math)
- Notes:
  - **DXTC1**: 2-color palette (or 3-color if color0 Γëñ color1); palette 11 = transparent (alpha=0) or opaque (alpha=0xff).
  - **DXTC3**: Explicit 4-bit alpha per pixel; color handling identical to DXTC1 but all colors fully opaque.
  - **DXTC5**: Interpolated 3-bit alpha indices; two alpha endpoints with 6 or 8 interpolated values depending on whether alpha0 > alpha1.
  - All iterate over 4├ù4 pixel blocks, unpack 16-bit and 32-bit values with endian swaps, and write as RGBA or BGRA depending on `ALEPHONE_LITTLE_ENDIAN`.

## Control Flow Notes
This file is part of the texture/resource loading pipeline (likely invoked during initialization or when maps/levels are loaded). It converts external DDS files into in-memory image descriptors usable by the renderer. The decompression functions are low-level helpers called by `MakeRGBA` to handle format conversions on demand. Data flows: file ΓåÆ DDS parsing ΓåÆ format detection ΓåÆ mipmap enumeration ΓåÆ per-level decompression ΓåÆ final RGBA or DXTC buffer in `ImageDescriptor`.

## External Dependencies
- **SDL** (`SDL.h`, `SDL_endian.h`): Surface creation and pixel blitting for format conversion.
- **OpenGL** (`gl.h`, `glu.h`, `OGL_Setup.h`): Conditional support for image scaling via `gluScaleImage` when GL is active.
- **Custom streams** (`AStream.h`): `AIStreamLE` for little-endian binary parsing of DDS header.
- **Type definitions** (`cstypes.h`): `uint32`, `uint16`, `uint8` integer types; `FOUR_CHARS_TO_INT` macro.
- **DDS structures** (`DDS.h`): `DDSURFACEDESC2` and surface descriptor flags (DDSD_*, DDPF_*, DDSCAPS_*, DDSCAPS2_*).
- **File handling** (`ImageLoader.h`, `FileHandler.h` via OpenedFile/FileSpecifier): File I/O abstraction.
- **Math**: Standard C++ `<cmath>` for `log2`, `floor`, `max`.
