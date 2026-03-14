# Source_Files/Misc/alephversion.h

## File Purpose
Defines compile-time version and platform identification macros for Aleph One (Marathon engine). Provides version strings, dates, and platform-specific identifiers used throughout the codebase for display and update logic.

## Core Responsibilities
- Define version display strings (`A1_DISPLAY_VERSION`, `A1_DISPLAY_DATE_VERSION`)
- Conditionally define platform display names and update platform codes based on OS detection
- Provide composite `A1_VERSION_STRING` macro combining version and platform info
- Support version synchronization across build system (noted comment references .r resource files)

## Key Types / Data Structures
None.

## Global / File-Static State
None (compile-time macros only; no runtime state).

## Key Functions / Methods
None.

## Control Flow Notes
This file participates only in compile-time preprocessing. Platform detection uses conditional compilation (`#ifdef`, `#elif`, `#else`) to select appropriate platform identifiers. The resulting macros are referenced elsewhere in the codebase for version displays and update checking logic (not visible in this file).

## External Dependencies
- **No includes**: Header is self-contained
- **Platform detection symbols** (not defined here): `WIN32`, `__APPLE__`, `__MACH__`, `__MACOS__`, `linux`, `__BEOS__`, `__NetBSD__`, `__OpenBSD__`, `PSP`

---

**Notes**: This is a configuration header, not a functional module. Version bump requires coordination with `Resources/Aleph One Classic SDL.r` as noted in the comment. The version appears frozen at 2008-07-21 (v0.20.2), suggesting this may be legacy code or a stable release branch.
