# config.h

## File Purpose
Autoconf-generated configuration header for the Aleph One SDL game engine. Defines preprocessor macros indicating which optional features and dependencies are available on this build system. Allows conditional compilation of subsystems like Lua scripting, OpenGL rendering, and various audio codecs.

## Core Responsibilities
- Declare feature availability via `HAVE_*` macros (used in conditional compilation throughout codebase)
- Define package metadata (name, version, bug report endpoint)
- Specify target platform ("elf mipsallegrexel" ΓÇö likely a PSP or embedded MIPS variant)
- Enable/disable optional subsystems: Lua, SDL extensions (image, net, font), audio decoders, profiling, Win32 music
- Standardize POSIX and C99 header availability detection

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This file is pure configuration consumed at compile time via `#include` guards and conditional compilation (`#ifdef HAVE_*`, etc.) in downstream source files.

## External Dependencies
- **Package metadata**: Aleph One v20080721, SDL port
- **Standard headers** (all declared HAVE_): `inttypes.h`, `stdint.h`, `stdlib.h`, `string.h`, `unistd.h`, `pwd.h`, `sys/types.h`, `sys/stat.h`
- **SDL libraries** (enabled): `SDL_image`, `SDL_net`, `SDL_ttf`
- **Audio libraries** (enabled): Ogg Vorbis (`vorbis/vorbisfile.h`); disabled: MAD MP3, SMPEG, Speex, libsndfile, ALSA
- **Scripting** (enabled): Lua
- **Graphics** (disabled): OpenGL
- **Build tools** (disabled): gprof profiling

---

**Notes**: Commented-out macros (`/* #undef ... */`) indicate features not available in this build. The platform target and SDL-heavy dependency profile suggest a portable handheld/embedded game engine configuration.
