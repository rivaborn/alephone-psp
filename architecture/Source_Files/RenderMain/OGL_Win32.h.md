# Source_Files/RenderMain/OGL_Win32.h

## File Purpose
Windows-specific header that provides dynamic function pointers for OpenGL ARB extensions (multitexturing and compressed textures). Supports runtime discovery and binding of these optional extensions through conditional compilation patterns.

## Core Responsibilities
- Define function pointer types for OpenGL ARB extensions
- Declare and conditionally export global function pointers
- Track multitexture support availability via a flag
- Provide macros that map extension function calls to dynamically resolved pointers
- Separate implementation initialization (`__OGL_Win32_cpp__`) from consumer code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GL_ACTIVETEXTUREARB_FUNC | typedef | Function pointer type for `glActiveTextureARB(unsigned int)` |
| GL_CLIENTACTIVETEXTUREARB_FUNC | typedef | Function pointer type for `glClientActiveTextureARB(int)` |

(PFNGLCOMPRESSEDTEXIMAGE2DARBPROC is predefined in `<GL/glext.h>`)

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| glActiveTextureARB_ptr | GL_ACTIVETEXTUREARB_FUNC | extern/global | Dynamically resolved function pointer; nullptr if unsupported |
| glClientActiveTextureARB_ptr | GL_CLIENTACTIVETEXTUREARB_FUNC | extern/global | Dynamically resolved function pointer; nullptr if unsupported |
| glCompressedTexImage2DARB_ptr | PFNGLCOMPRESSEDTEXIMAGE2DARBPROC | extern/global | Dynamically resolved function pointer; nullptr if unsupported |
| has_multitex | int | extern/global | Boolean flag: 1 if multitexturing extensions available, 0 otherwise |

## Key Functions / Methods
### setup_gl_extensions
- Signature: `extern void setup_gl_extensions()`
- Purpose: Dynamically load and bind OpenGL ARB extension function pointers at runtime
- Notes: Implementation is in `OGL_Win32.cpp`; must be called during initialization to populate the function pointers

## Control Flow Notes
Part of the **initialization phase**. `setup_gl_extensions()` is called once at engine startup to dynamically resolve function pointers via platform-specific mechanisms (likely `wglGetProcAddress` on Windows). The three function pointers start as null and are populated during this setup phase. Consumers then use the conditional macros to call these extensions transparently.

## External Dependencies
- `<GL/gl.h>` ΓÇö OpenGL core headers
- `<GL/glext.h>` ΓÇö OpenGL extensions; provides `PFNGLCOMPRESSEDTEXIMAGE2DARBPROC`
- Implementation file: `OGL_Win32.cpp` (defines `setup_gl_extensions()`)
