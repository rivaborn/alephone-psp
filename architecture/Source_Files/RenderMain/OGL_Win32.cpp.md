# Source_Files/RenderMain/OGL_Win32.cpp

## File Purpose
Initializes OpenGL ARB extensions on Windows by dynamically loading extension function pointers via SDL. Detects and enables multitexturing capability and compressed texture support for the renderer.

## Core Responsibilities
- Dynamically load OpenGL extension function pointers using `SDL_GL_GetProcAddress`
- Detect availability of ARB multitexturing extensions
- Initialize compressed texture extension pointer
- Set global flag to indicate multitexture support status
- Report missing extensions to stderr

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GL_ACTIVETEXTUREARB_FUNC | typedef | Function pointer signature for `glActiveTextureARB` (sets active texture unit) |
| GL_CLIENTACTIVETEXTUREARB_FUNC | typedef | Function pointer signature for `glClientActiveTextureARB` (sets active client texture unit) |
| PFNGLCOMPRESSEDTEXIMAGE2DARBPROC | typedef | Standard OpenGL function pointer type for compressed 2D texture image upload |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| glActiveTextureARB_ptr | GL_ACTIVETEXTUREARB_FUNC | global (extern'd) | Pointer to runtime-loaded `glActiveTextureARB` function |
| glClientActiveTextureARB_ptr | GL_CLIENTACTIVETEXTUREARB_FUNC | global (extern'd) | Pointer to runtime-loaded `glClientActiveTextureARB` function |
| glCompressedTexImage2DARB_ptr | PFNGLCOMPRESSEDTEXIMAGE2DARBPROC | global (extern'd) | Pointer to runtime-loaded `glCompressedTexImage2DARB` function |
| has_multitex | int | global (extern'd) | Flag: 1 if multitexture ARB extensions available, 0 otherwise |

## Key Functions / Methods

### setup_gl_extensions
- **Signature:** `void setup_gl_extensions()`
- **Purpose:** Detect and initialize OpenGL ARB extension function pointers at runtime.
- **Inputs:** None
- **Outputs/Return:** None (modifies global state)
- **Side effects:** 
  - Sets `glActiveTextureARB_ptr`, `glClientActiveTextureARB_ptr`, `glCompressedTexImage2DARB_ptr` globals
  - Sets `has_multitex` flag (1 if both active-texture extensions found, 0 otherwise)
  - Writes to stderr if `glCompressedTexImage2DARB` not available
  - I/O: Calls SDL and OpenGL runtime functions
- **Calls:** `SDL_GL_GetProcAddress` (3 times)
- **Notes:** 
  - Multitexture detection requires both `glActiveTextureARB` AND `glClientActiveTextureARB` to be present
  - Missing compressed texture extension is logged but non-fatal
  - Should be called once during renderer initialization before rendering begins

## Control Flow Notes
This is a startup/initialization function. It appears to be called early in the rendering pipeline setup (likely during engine init) to ensure extension pointers are loaded before any rendering code attempts to use multitexturing or compressed textures. The global flag `has_multitex` is likely checked elsewhere to conditionally enable multitexture rendering paths.

## External Dependencies
- **windows.h** ΓÇö Windows API (platform target)
- **GL/gl.h, GL/glext.h** ΓÇö OpenGL core and extension definitions
- **SDL.h** ΓÇö SDL library for cross-platform GL proc address loading
- **OGL_Win32.h** ΓÇö Local header defining function pointer types and extern declarations
