# Source_Files/GameWorld/effect_definitions.h

## File Purpose
Defines effect metadata structures and a table of effect definitions for the game. Effects are visual/audio animations triggered by game events (projectile impacts, entity deaths, splashes, etc.). This header centralizes effect configuration for the Aleph One game engine (Marathon-style).

## Core Responsibilities
- Define effect behavior flags (`_end_when_animation_loops`, `_sound_only`, `_media_effect`, etc.)
- Define the `effect_definition` struct encapsulating effect metadata
- Declare a mutable static array of effect definitions
- Provide a const table of ~60 predefined effect configurations mapped to game events
- Declare serialization functions for effect definitions (pack/unpack from streams)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `effect_definition` | struct | Metadata for an effect: collection/shape indices, sound pitch, behavior flags, animation/sound delays |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `effect_definitions` | `effect_definition[]` | static | Mutable array; effect definitions can be loaded/modified at runtime |
| `original_effect_definitions` | `const effect_definition[]` | static | Default effect configurations for explosions, splashes, ricochets, teleports, etc. |

## Key Functions / Methods

### unpack_effect_definition
- Signature: `uint8 *unpack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Deserialize effect definitions from a binary stream (used by 1-2-3 Converter)
- Inputs: byte stream, destination array, count of objects to unpack
- Outputs/Return: updated stream pointer
- Side effects: modifies Objects array; advances stream pointer
- Calls: Not inferable from this file
- Notes: Declaration only; implementation elsewhere

### pack_effect_definition
- Signature: `uint8 *pack_effect_definition(uint8 *Stream, effect_definition *Objects, size_t Count)`
- Purpose: Serialize effect definitions to a binary stream (used by 1-2-3 Converter)
- Inputs: byte stream, source array, count of objects to pack
- Outputs/Return: updated stream pointer
- Side effects: writes to stream; advances stream pointer
- Calls: Not inferable from this file
- Notes: Declaration only; implementation elsewhere

## Control Flow Notes
Effect definitions are looked up at initialization and during gameplay to spawn animations when game events occur (weapon fires, entities take damage, media interactions). The static array allows runtime updates to effect behavior; const defaults provide sensible fallbacks.

## External Dependencies
- **Collection macros** (`_collection_rocket`, `_collection_fighter`, `_collection_compiler`, etc.): define sprite/animation libraries
- **Frequency constants** (`_normal_frequency`, `_higher_frequency`, `_lower_frequency`): sound pitch values
- **BUILD_COLLECTION()** macro: constructs collection identifiers
- **Sound enum** (`_snd_teleport_in`): sound effect IDs
- **Constants** (`NUMBER_OF_EFFECT_TYPES`, `TICKS_PER_SECOND`, `NONE`): indices and time values
- All defined elsewhere in codebase

---

**Note:** Line 2 contains a typo in the include guard (`__EFFECT_DEFINTIIONS_H` should be `__EFFECT_DEFINITIONS_H`), but the actual guard on line 1 is spelled correctly, so the header will include properly.
