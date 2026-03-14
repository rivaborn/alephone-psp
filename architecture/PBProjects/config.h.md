# PBProjects/config.h

## File Purpose
Auto-generated configuration header produced by Autoconf's `configure` script. Defines compile-time feature flags and package metadata to conditionally include/exclude optional dependencies (OpenGL, SDL extensions) based on the build environment. This is included by source files to adapt compilation.

## Core Responsibilities
- Define `HAVE_*` feature availability macros for optional libraries
- Enable/disable OpenGL support via `HAVE_OPENGL`
- Enable/disable SDL extensions (image/networking) via `HAVE_SDL_IMAGE`, `HAVE_SDL_NET`
- Declare corresponding header availability macros
- Store package name ("AlephOne") and version ("0.13.0")
- Control X11/graphics platform selection via `X_DISPLAY_MISSING`

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
This file has no runtime control flow. It is a pure configuration header included at compile time. Source files test `#ifdef HAVE_OPENGL`, `#ifdef HAVE_SDL_IMAGE`, etc., to conditionally compile graphics, image loading, and networking code paths.

## External Dependencies
- **No includes**: This is a generated header; it defines symbols but includes no other files.
- **Consumed by**: Any source file needing to detect optional feature support.

---

**Notes:** This is Autoconf output (as indicated by the "Generated automatically by configure" comment). The defines show that OpenGL and both SDL extension libraries (image + networking) are enabled in this build configuration. The build system appears to be targeting a Unix-like environment with SDL as the multimedia backend (evidenced by SDL_image.h, SDL_net.h, and unistd.h availability).
