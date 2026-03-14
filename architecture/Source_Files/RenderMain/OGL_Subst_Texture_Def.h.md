# Source_Files/RenderMain/OGL_Subst_Texture_Def.h

## File Purpose
Defines substitute texture options and management functions for OpenGL rendering of walls and sprites in the Aleph One engine. This header specifies how custom textures (loaded from image files) replace the original in-game graphics, including positioning and transparency behavior.

## Core Responsibilities
- Define extended texture options (`OGL_TextureOptions`) for sprite and wall substitutions
- Provide sprite positioning parameters (scaling, offset calculation)
- Expose texture collection management (load, unload, count)
- Support XML-based texture configuration parsing
- Handle image positioning calculation relative to original bitmaps

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| OGL_TextureOptions | struct | Extends base texture options with sprite-specific positioning (offset, scaling) and void visibility control |

## Global / File-Static State
None.

## Key Functions / Methods

### OGL_GetTextureOptions
- Signature: `OGL_TextureOptions *OGL_GetTextureOptions(short Collection, short CLUT, short Bitmap)`
- Purpose: Retrieve the texture options currently configured for a specific bitmap
- Inputs: Collection ID, CLUT (color lookup table) index, Bitmap index
- Outputs/Return: Pointer to configured OGL_TextureOptions struct
- Side effects: None inferable from header
- Calls: Not inferable from this file
- Notes: Used to query existing texture configurations per collection/bitmap

### OGL_CountTextures
- Signature: `int OGL_CountTextures(short Collection)`
- Purpose: Report the total number of textures in a collection
- Inputs: Collection ID
- Outputs/Return: Integer count of textures
- Side effects: None inferable
- Calls: Not inferable from this file
- Notes: Used for texture enumeration or validation

### OGL_LoadTextures / OGL_UnloadTextures
- Signature: `void OGL_LoadTextures(short Collection)`, `void OGL_UnloadTextures(short Collection)`
- Purpose: Load or unload all substitute textures for a collection (rendering prep/cleanup)
- Inputs: Collection ID
- Outputs/Return: None
- Side effects: GPU memory allocation/deallocation, image file I/O
- Calls: Not inferable from this file
- Notes: Likely called during level load/unload or resource management

### OGL_TextureOptions::FindImagePosition / Original
- Signature: `void FindImagePosition()`, `void Original()`
- Purpose: Calculate sprite corner positions (Right, Bottom) from Left/Top offsets and image size; `Original()` resets to defaults
- Inputs: None (uses member variables: Left, Top, ImageScale, and image dimensions)
- Outputs/Return: None (modifies member state)
- Side effects: Modifies Left, Top, Right, Bottom members
- Notes: Positioning is in internal units (world_unit=1024 per pixel)

## Control Flow Notes
This is a definition header; initialization/management likely occurs in a corresponding `.cpp` implementation. Texture loading/unloading integrates with the engine's resource management lifecycle (level load ΓåÆ load textures ΓåÆ render ΓåÆ unload textures). XML parsing support (`TextureOptions_GetParser()`) suggests configuration from MML (Marathon Markup Language) files.

## External Dependencies
- `OGL_Texture_Def.h`: Base texture options struct and enums (opacity types, blend types, ImageDescriptor)
- `XML_ElementParser.h`: XML parsing framework for configuration
- Conditional compilation: `#ifdef HAVE_OPENGL` (entire file)
