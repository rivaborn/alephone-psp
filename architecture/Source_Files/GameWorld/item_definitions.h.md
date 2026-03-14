# Source_Files/GameWorld/item_definitions.h

## File Purpose
Defines the data structure and lookup table for all in-game items (weapons, ammunition, powerups, keys, game balls). Provides metadata mapping for item behavior, display names, visual shapes, and spawn limits. Part of the Aleph One/Marathon engine's item system.

## Core Responsibilities
- Define `item_definition` struct to describe item properties
- Provide static initialization table for all 40+ item types in the game
- Map item kinds to singular/plural name IDs for localization
- Associate items with their shape descriptors for rendering
- Enforce per-player item limits (e.g., max 2 pistols, max 1 fusion pistol)
- Flag items with environment restrictions (vacuum-dangerous weapons)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `item_definition` | struct | Describes a single item's metadata: kind, names, shape, limits, restrictions |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `item_definitions[]` | `struct item_definition[]` | static | Lookup table indexed by item ID; contains ~40 entries covering weapons, ammo, powerups, keys, and net game balls |

## Key Functions / Methods
NoneΓÇöthis is purely data-driven initialization.

## Control Flow Notes
Static data loaded at engine initialization. Accessed during:
- **Item spawning**: to fetch shape descriptor and environment restrictions
- **Player pickup logic**: to validate inventory limits and apply effects
- **UI/localization**: to retrieve singular/plural name IDs

Indexed lookup suggests O(1) access pattern for item metadata queries during gameplay.

## External Dependencies
- **Macros**: `BUILD_DESCRIPTOR()`, `BUILD_COLLECTION()` (defined elsewhereΓÇöshape builder macros)
- **Enums** (defined elsewhere):
  - Item kinds: `_weapon`, `_ammunition`, `_powerup`, `_item`, `_ball`
  - Special values: `UNONE`, `NONE`
  - Environments: `_environment_vacuum`, `_environment_single_player`
- **Type**: `shape_descriptor` (defined elsewhereΓÇölikely a collection/shape reference)

**Notes**: 
- The conditional guard `#ifndef DONT_REPEAT_DEFINITIONS` suggests this table is also mirrored in `script_instructions.cpp` (Pfhortran scripting).
- Items without visual representation use `UNONE` for shape.
- Powerups use `NONE` for both singular and plural name IDs (likely hardcoded visuals with no text labels).
- Vacuum-restricted items cannot spawn in airless environments.
