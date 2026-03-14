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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ImageDescriptor` | class | Image descriptor that holds pixel data, dimensions, and format info |
| `FileSpecifier` | class | File abstraction; encapsulates path and file I/O operations |
| `SDL_Surface` | struct | SDL's representation of in-memory pixel data with format metadata |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadFromFile
- **Signature:** `bool ImageDescriptor::LoadFromFile(FileSpecifier& File, int ImgMode, int flags, int actual_width = 0, int actual_height = 0, int maxSize = 0)`
- **Purpose:** Load an image file and convert it to RGBA format (either color or opacity channel).
- **Inputs:**
  - `File`: FileSpecifier pointing to the image file to load
  - `ImgMode`: Load mode; either `ImageLoader_Colors` (RGB) or `ImageLoader_Opacity` (grayscale-to-alpha)
  - `flags`: Bitset of loader options (e.g., `ImageLoader_ResizeToPowersOfTwo`, `ImageLoader_ImageIsAlreadyPremultiplied`)
  - `actual_width`, `actual_height`: Original image dimensions for UV scaling; 0 means use loaded dimensions
  - `maxSize`: Maximum allowed texture size (parameter present but not used in visible code)
- **Outputs/Return:** `true` on success; `false` if file cannot be loaded or dimensions mismatch
- **Side effects:**
  - Allocates or resizes pixel buffer via `Resize()` (uses `new[]`)
  - Modifies member variables: `Width`, `Height`, `VScale`, `UScale`, `MipMapCount`, `PremultipliedAlpha`
  - Frees SDL surfaces via `SDL_FreeSurface()`
- **Calls:**
  - `LoadDDSFromFile()`: Attempts DDS format load if Colors mode
  - `IsPresent()`: Checks if pixel data already exists
  - `IMG_Load()` or `SDL_LoadBMP()`: SDL image loading (fallback if SDL_image unavailable)
  - `SDL_CreateRGBSurface()`: Create RGBA surface (endianness-aware)
  - `SDL_SetAlpha()`, `SDL_BlitSurface()`: Convert and copy image data
  - `Resize()`: Allocate pixel buffer to specified dimensions
  - `GetPixelBasePtr()`: Access internal pixel array
  - `NextPowerOfTwo()`: Round dimensions up to power of 2 (if flag set)
  - `PIN()`: Clamp opacity value to [0, 255]
- **Notes:**
  - Handles both little-endian (`0x000000ff` RGBA) and big-endian (`0xff000000` RGBA) platforms.
  - For **Colors mode**: copies all 32-bit RGBA pixels directly via `memcpy()`.
  - For **Opacity mode**: converts RGB to grayscale (average of R, G, B) and stores in the alpha channel (byte 3 of each pixel); requires color image already loaded (validates dimensions match).
  - DDS loading is attempted first if Colors mode; falls through to SDL image loading if DDS returns false.
  - SDL_SetAlpha disables per-pixel alpha blending during blit to ensure raw pixel copy.

## Control Flow Notes
This function is called during asset loading (not per-frame). It is part of the image-descriptor initialization pipeline:
1. **Pre-load check**: If loading opacity and no color image exists, return false.
2. **DDS attempt**: Try to load as compressed DDS format (if Colors mode).
3. **SDL load**: Load image file using SDL_image or BMP fallback.
4. **Dimension processing**: Read loaded dimensions; optionally round to power-of-two.
5. **Allocation**: Resize internal buffer (Colors mode) or validate existing buffer (Opacity mode).
6. **Format conversion**: Create intermediary RGBA surface, blit loaded image, convert to internal format.
7. **Cleanup**: Free SDL surfaces and return status.

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
