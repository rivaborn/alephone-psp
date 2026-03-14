# PBProjects/precompiled_headers.h

## File Purpose
Precompiled header file for the Aleph One game engine (OSX variant) to optimize build times. Conditionally includes SDL libraries and macOS Carbon API headers, along with core game engine headers, to be compiled once and reused across translation units.

## Core Responsibilities
- Conditionally include SDL multimedia libraries (cross-platform support)
- Define macOS API target constants (Carbon/OS8/OSX selection)
- Aggregate includes for game engine core modules (types, rendering, world simulation, networking, sound)
- Compiler optimization via precompilation (GCC >= 3 guard)

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TARGET_API_MAC_CARBON` | Preprocessor macro | global | Set to 1 when `mac` is defined; signals macOS Carbon API usage |
| `SDL`, `mac`, `__GNUG__`, `__GNUC__` | Preprocessor conditionals | global | Feature and platform detection gates |

## Key Functions / Methods
None.

## Control Flow Notes
This file is processed during compilation phase only (not runtime). Preprocessor guards enforce platform-specific behavior:
- If `SDL` defined and not Mac: includes SDL multimedia libraries
- If `mac` defined: undefines conflicting macros, defines `TARGET_API_MAC_CARBON`, imports Carbon framework
- If `__GNUC__ >= 3`: includes all core engine headers (types, rendering, world, scripting, networking, sound, etc.)

## External Dependencies
- **SDL libraries**: `SDL.h`, `SDL_net.h`, `SDL_image.h` (cross-platform multimedia)
- **macOS frameworks**: `Carbon/Carbon.h` (native macOS API)
- **Game engine headers** (defined elsewhere): `cstypes.h`, `map.h`, `shell.h`, `render.h`, `OGL_Render.h`, `FileHandler.h`, `wad.h`, `preferences.h`, `scripting.h`, `network.h`, `mysound.h`, `csmacros.h`, `Model3D.h`, `dynamic_limits.h`, `effects.h`, `monsters.h`, `world.h`, `player.h`
