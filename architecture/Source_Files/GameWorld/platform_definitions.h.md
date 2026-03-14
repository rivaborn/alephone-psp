# Source_Files/GameWorld/platform_definitions.h

## File Purpose
Defines static configuration data for all platform types in the game world (doors and moving platforms). Initializes a lookup table mapping each platform type to its audio effects, movement properties, behavioral flags, and damage parameters.

## Core Responsibilities
- Define sound code constants for platform event types (starting, stopping, obstructed, uncontrollable)
- Declare the `platform_definition` structure that bundles audio, defaults, and damage data
- Populate static array `platform_definitions[]` with configuration for each platform type (8 types total)
- Provide indexed access to platform behavior at initialization and runtime

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `platform_definition` | struct | Groups audio event IDs, default platform behavior flags, and damage properties for a single platform type |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `platform_definitions` | `struct platform_definition[NUMBER_OF_PLATFORM_TYPES]` | static | Lookup table indexed by platform type, defines all properties for each platform kind |

## Key Functions / Methods
None. This file contains only data definitions and constants.

## Notes
- Eight platform types defined: three SPHT doors (basic, split, locked), basic SPHT platform, noisy SPHT platform, heavy SPHT door, Pfhor door, heavy SPHT platform, Pfhor platform
- Sound layout: extension/contraction sounds for start/stop, obstruction and uncontrollable sounds, ambient sound
- Each entry uses `FLAG()` macro to combine boolean behavior flags (e.g., `_platform_is_player_controllable`, `_platform_comes_from_ceiling`)
- Damage configuration varies: most use `_damage_crushing` with different intensity parameters
- Key item index is `NONE` for all defined platforms (not player-collectible)

## Control Flow Notes
This is a data definition file used at engine initialization. The `platform_definitions` array is indexed by a platform's type ID to configure its behavior, sounds, and damage properties during platform instantiation and simulation.

## External Dependencies
- **Defined elsewhere**: `static_platform_data` (struct), `damage_definition` (struct), sound constants (`_snd_*`), platform type constants (`_platform_is_*`), behavior flag constants, `NUMBER_OF_PLATFORM_TYPES` macro, `FLAG()` macro, `NONE` constant, `FIXED_ONE` constant
