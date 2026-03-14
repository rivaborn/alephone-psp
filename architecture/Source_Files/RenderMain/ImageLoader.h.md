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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class | Encapsulates loaded image data (pixels, dimensions, format, mipmaps, UV scaling) |
| `ImageDescriptor::ImageFormat` | enum | Pixel formats: RGBA8, DXTC1, DXTC3, DXTC5, Unknown |
| `copy_on_edit<T>` | template class | Lazy copy wrapper; read from original, create copy on edit |
| `ImageDescriptorManager` | typedef | Alias for `copy_on_edit<ImageDescriptor>` |

## Global / File-Static State
None.

## Key Functions / Methods

### ImageDescriptor::LoadFromFile
- **Signature:** `bool LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- **Purpose:** Load image from disk file (main entry point for image loading)
- **Inputs:** FileSpecifier reference, image mode (Colors/Opacity), flags (resize, DXTC, mipmaps, etc.), optional dimension overrides, max size limit
- **Outputs/Return:** bool (success/failure)
- **Side effects:** Allocates pixel buffer; sets Width, Height, Size, Format, MipMapCount, possibly PremultipliedAlpha
- **Calls:** `LoadDDSFromFile()` (delegates DDS format handling)
- **Notes:** ImgMode selects whether to load color or opacity channel; flags control processing (resize to POT, enable DXTC, load mipmaps)

### ImageDescriptor::Resize (dimension overload)
- **Signature:** `void Resize(int _Width, int _Height)`
- **Purpose:** Reallocate and resize image to new dimensions
- **Inputs:** New width and height
- **Outputs/Return:** void
- **Side effects:** Deallocates old Pixels, allocates new buffer, updates Width, Height, Size
- **Calls:** None visible
- **Notes:** Does not preserve existing pixel content; intended for fresh allocation

### ImageDescriptor::Resize (with size overload)
- **Signature:** `void Resize(int _Width, int _Height, int _TotalBytes)`
- **Purpose:** Resize image with explicit total buffer byte count (for mipmaps)
- **Inputs:** New width, height, total byte allocation
- **Outputs/Return:** void
- **Side effects:** Reallocates Pixels buffer; updates Width, Height, Size
- **Calls:** None visible
- **Notes:** Used when buffer contains multiple mipmap levels

### ImageDescriptor::Minify
- **Signature:** `bool Minify()`
- **Purpose:** Reduce image resolution by half (for mipmap generation)
- **Inputs:** None
- **Outputs/Return:** bool (success)
- **Side effects:** Modifies Pixels buffer (downsamples); updates Width, Height, Size
- **Calls:** None visible
- **Notes:** Likely implements 2├ù2 box filter downsampling

### ImageDescriptor::MakeRGBA / MakeDXTC3
- **Signature:** `bool MakeRGBA()` / `bool MakeDXTC3()`
- **Purpose:** Convert pixel data between formats
- **Inputs:** None
- **Outputs/Return:** bool (success)
- **Side effects:** Reallocates Pixels, changes Format field
- **Calls:** None visible
- **Notes:** Format conversion; may involve recompression for DXTC

### ImageDescriptor::PremultiplyAlpha
- **Signature:** `void PremultiplyAlpha()`
- **Purpose:** Premultiply RGB channels by alpha channel
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies Pixels in-place; sets PremultipliedAlpha flag
- **Calls:** None visible
- **Notes:** Required for correct blending in some rendering pipelines

### ImageDescriptor::GetMipMapPtr
- **Signature:** `uint32 *GetMipMapPtr(int Level)` / `const uint32 *GetMipMapPtr(int Level) const`
- **Purpose:** Retrieve pointer to mipmap data at given detail level
- **Inputs:** Mipmap level (0 = full resolution)
- **Outputs/Return:** Pointer to mipmap pixel data (mutable or const)
- **Side effects:** None
- **Calls:** None visible
- **Notes:** Offsets into the Pixels buffer; assumes all mipmaps are stored contiguously

### copy_on_edit<T>::get / edit
- **Signature:** `const T* get()` / `T* edit()` / `T* edit(T* copy)`
- **Purpose:** Access (read-only) or modify (copy-on-edit) the managed resource
- **Inputs:** Optional copy pointer for pre-made copy
- **Outputs/Return:** Pointer to resource (const or mutable)
- **Side effects:** `edit()` creates a copy of `_original` if one does not exist; takes ownership
- **Calls:** None visible
- **Notes:** `get()` always returns valid pointer (copy or original); `edit()` creates copy lazily only when needed for modification

### Global Utility
**GetMipMapPtr** ΓÇö `uint32 *GetMipMapPtr(uint32 *pixels, int size, int level, int width, int height, int format)`
- Calculate mipmap pointer within a buffer given dimensions and format; utility for manual mipmap traversal outside ImageDescriptor

## Control Flow Notes
ImageDescriptor objects are typically created and populated via `LoadFromFile()` during asset loading (engine initialization or streaming). Once loaded, the object serves as read-mostly storage for rendering systems. The `copy_on_edit` wrapper is used to defer expensive copies until a modification is actually needed. Private methods (`LoadDDSFromFile`, `LoadMipMapFromFile`) handle file I/O during load; public dimension/format accessors serve rendering pipelines during frame rendering.

## External Dependencies
- **DDS.h:** `DDSURFACEDESC2` struct (DirectDraw Surface format descriptor)
- **cseries.h:** Platform abstractions, type definitions
- **FileHandler.h:** `FileSpecifier`, `OpenedFile` classes for file I/O
- **\<vector\>:** STL vector (included but not directly used in this header; possibly for implementation)
- **uint32, uint8, uint16** ΓÇö Fixed-width integer types (assumed from cstypes.h)
