# Subsystem Overview

## Purpose

PBProjects is the build configuration and platform abstraction layer for Aleph One. It provides Autoconf-based compile-time feature detection (optional libraries like OpenGL and SDL extensions), platform-specific path configuration, precompiled header aggregation for build optimization, and macOS Cocoa GUI integration.

## Key Files

| File | Role |
|------|------|
| `config.h` | Auto-generated Autoconf configuration header; defines `HAVE_*` macros for optional features (OpenGL, SDL_image, SDL_net) |
| `confpaths.h` | Compile-time configuration paths for data file locations (macOS .app bundle structure) |
| `precompiled_headers.h` | PCH file aggregating SDL and macOS framework includes plus game engine headers for build optimization |
| `SDLMain.h` | Objective-C Cocoa application controller; declares menu action handlers for game lifecycle and preferences |

## Core Responsibilities

- Auto-generate and maintain compile-time feature availability macros consumed by source files
- Define platform-specific resource and data directory paths
- Aggregate frequently-included headers into a precompiled header for compiler optimization
- Provide macOS Cocoa GUI entry point with menu-driven game lifecycle handlers (new/open/save game, preferences, help)
- Abstract platform differences through conditional compilation (`X_DISPLAY_MISSING`, platform-specific Carbon/OSX macros)

## Key Interfaces & Data Flow

**Exposes:**
- `config.h` feature macros (`HAVE_OPENGL`, `HAVE_SDL_IMAGE`, `HAVE_SDL_NET`) to any source file requiring optional feature detection
- `SDLMain.h` menu action methods for game lifecycle control via Cocoa responder chain
- Precompiled headers for reduced compilation time across translation units

**Consumes:**
- SDL multimedia libraries (core, image, networking)
- macOS frameworks (Cocoa, Carbon)
- Game engine module headers (render, sound, network, scripting, world simulation)

## Runtime Role

- `config.h`: Compile-time conditional compilation only; no runtime role
- `SDLMain.h`: Serves as primary macOS GUI entry point during application initialization; dispatches menu-driven user actions
- `precompiled_headers.h`: Build-time optimization; no runtime role

## Notable Implementation Details

- `config.h` is Autoconf-generated output; shows OpenGL and both SDL extension libraries enabled in this build configuration
- `confpaths.h` references `.app` bundle paths, indicating macOS-specific resource layout expectations
- `precompiled_headers.h` includes GCC >= 3 guard, suggesting compatibility with older compiler toolchains
- `SDLMain.h` uses Objective-C (`NSObject` base class from Foundation/Cocoa) for native macOS GUI integration
