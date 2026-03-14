# Source_Files/RenderMain/DDS.h

## File Purpose
Defines DirectDraw Surface (DDS) file format structures and constants for parsing DDS image files. Part of the Aleph One game engine's rendering subsystem. Based on Microsoft's DirectX 9 DDS file format reference.

## Core Responsibilities
- Define DDS surface descriptor flags (`DDSD_*`) for validating which header fields are present
- Define pixel format flags (`DDPF_*`) to identify color formats (RGB, FourCC, alpha)
- Define surface capability flags (`DDSCAPS_*`, `DDSCAPS2_*`) for texture properties (mipmaps, cubemaps, volumes)
- Declare `DDSURFACEDESC2` structure representing the complete DDS file header
- Provide a portable definition independent of native DirectDraw headers (guarded by `__DDRAW_INCLUDED__`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| DDSURFACEDESC2 | struct | Complete DDS surface descriptor; matches binary file header format for direct memory mapping |
| ddpfPixelFormat | struct (nested within DDSURFACEDESC2) | Pixel format with color channel bit masks, FourCC code, and bit-per-pixel count |
| ddsCaps | struct (nested within DDSURFACEDESC2) | Surface capabilities flags for texture properties (mipmap levels, cubemap faces, volume textures) |

## Global / File-Static State
None.

## Key Functions / Methods
None (header-only structure definitions).

## Control Flow Notes
Not applicable. This is a data definition file with no control flow. Used by DDS file loaders elsewhere in the rendering pipeline to validate and parse DDS file headers.

## External Dependencies
- `cstypes.h` ΓÇö provides platform-independent `uint32` type
- Conditional guard `__DDRAW_INCLUDED__` to avoid conflicts with native DirectDraw SDK headers

## Notes
- All `#define` constants are bit flags used with bitwise operations for feature detection
- `DDSURFACEDESC2` struct layout mirrors the binary DDS file header for direct serialization/deserialization
- `dwReserved1[11]` and `dwReserved2` fields maintain binary compatibility with DirectX specification
- File conditionally compiles only if native DirectDraw headers are not present
- Written by Gregory Smith in 2005 for Aleph One project
