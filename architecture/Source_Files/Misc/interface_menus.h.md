# Source_Files/Misc/interface_menus.h

## File Purpose
Defines menu and menu-item identifiers for the Aleph One game engine's UI. Provides constants for in-game menus (pause, save, quit) and interface/main menus (new game, load, preferences), plus CoreFoundation menu names for macOS NIB-based menu construction.

## Core Responsibilities
- Define enum constants for game menu ID and item IDs
- Define enum constants for interface/main menu ID and item IDs
- Declare a fake empty menu to suppress empty menu bar rendering at exit
- Provide menu name strings for macOS NIB resource loading

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mGame` enum | enum (constants) | Menu ID and item identifiers available during active gameplay |
| `mInterface` enum | enum (constants) | Menu ID and item identifiers for the main interface/menu screen |
| `mFakeEmptyMenu` enum | enum (constants) | Placeholder menu to prevent empty menu bar display |
| `MenuNames` | `CFStringRef[]` | Array of menu names for macOS NIB integration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `NumMenus` | const int | Global | Count of menus (always 3) |
| `MenuNames` | const CFStringRef[] | Global | Menu identifiers for macOS NIB resource loading; insertion order is reversed (Fake_Empty, Interface, Game) |

## Key Functions / Methods
None.

## Control Flow Notes
Not applicable. This is a configuration header containing only constants and enumerations. Used by higher-level menu management code (likely `interface_macintosh.cpp` per comments) during initialization and menu bar construction.

## External Dependencies
- **CoreFoundation** (macOS): `CFStringRef`, `CFSTR()` macro for wide string literals
- **Conditional compilation**: `USES_NIBS` flag gates macOS-specific menu name definitions
- **Aleph One engine**: Part of Marathon-engine port; defines IDs referenced elsewhere in codebase

## Notes
- Menu IDs (128ΓÇô130) are hardcoded to reserved ranges, typical of classic Mac menu architecture
- Reverse insertion order comment suggests dependency on macOS menu bar construction order
- Fake empty menu (ID 130) is a workaround to hide empty menu bars at application exit
