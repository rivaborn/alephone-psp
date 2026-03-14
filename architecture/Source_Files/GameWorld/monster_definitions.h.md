# Source_Files/GameWorld/monster_definitions.h

## File Purpose
Defines all monster types and their behavioral/physical attributes for the game engine. Contains data structures for monster definitions and attack parameters, along with initialization data for ~20+ distinct monster types, each with minor/major variants and special forms (invisible, kamikaze, tiny, etc.).

## Core Responsibilities
- Define monster class enumerations and class relationship macros (friends/enemies via bitfields)
- Define behavioral flags (omniscient, invisible, kamikaze, berserker, etc.)
- Define speed, intelligence, and door-handling difficulty constants
- Define `attack_definition` struct for melee and ranged attack parameters
- Define `monster_definition` struct (~128 bytes) encapsulating all monster attributes
- Declare global mutable and immutable monster definition arrays
- Provide serialization helpers for saving/loading monster definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `attack_definition` | struct | Single attack template: projectile type, repetitions, aiming error, range, animation frame, spawn offset (dx/dy/dz) |
| `monster_definition` | struct | Complete monster template: vitality, immunities, weaknesses, flags, class affiliation, AI properties (intelligence, speed, visual range), physics (gravity, height, radius), shapes, sounds, melee/ranged attacks |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `monster_definitions` | `struct monster_definition[]` | global | Mutable array of all active monster definitions; indexed by monster type ID |
| `original_monster_definitions` | `const struct monster_definition[]` | static | Read-only initial values for all monster types; used as defaults |

## Key Functions / Methods

### unpack_monster_definition
- **Signature:** `uint8 *unpack_monster_definition(uint8 *Stream, monster_definition *Objects, size_t Count)`
- **Purpose:** Deserialize monster definitions from a binary stream (e.g., loading from save file or resource)
- **Inputs:** Byte stream pointer, target monster definition array, count of definitions to unpack
- **Outputs/Return:** Updated stream pointer (post-read position)
- **Side effects:** Modifies the Objects array in-place
- **Calls:** Not visible in this file (defined in source)
- **Notes:** Enables save/load of custom monster definition changes

### pack_monster_definition
- **Signature:** `uint8 *pack_monster_definition(uint8 *Stream, monster_definition *Objects, size_t Count)`
- **Purpose:** Serialize monster definitions to a binary stream
- **Inputs:** Byte stream pointer, monster definition array, count of definitions to pack
- **Outputs/Return:** Updated stream pointer (post-write position)
- **Side effects:** Writes to the stream buffer
- **Calls:** Not visible in this file (defined in source)
- **Notes:** Inverse of `unpack_monster_definition`

## Control Flow Notes
This is a **pure data definition file**, not part of the main game loop. Monster definitions are loaded once during initialization and are read-only during gameplay (except for dynamic modifications). The `monster_definitions[]` array is indexed by monster type ID and referenced by AI/behavior systems to determine attack patterns, physics, speeds, and visual properties. The presence of both mutable and immutable arrays suggests the engine supports resetting monsters to defaults or allowing runtime tweaks.

## External Dependencies
- **Includes:** `effects.h` (for effect type enums like `_effect_fighter_blood_splash`), `projectiles.h` (for projectile type enums like `_projectile_staff`)
- **Defined elsewhere:** 
  - Macros: `WORLD_ONE`, `WORLD_ONE_HALF`, `WORLD_ONE_FOURTH`, `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `NUMBER_OF_ANGLES`, `BUILD_COLLECTION()`, `FLAG()`, `NONE`, `UNONE`
  - Types: `int16`, `uint32`, `uint8`, `_fixed`, `angle`, `world_distance`, `shape_descriptor`, `damage_definition`
  - Constants: `NUMBER_OF_MONSTER_TYPES`, effect/projectile type enums
  - Sounds: `_snd_fighter_activate`, `_snd_human_wail`, etc. (external sound IDs)

**Notes:**  
The file extensively uses C-style designated initializer syntax (comments labeling each field in aggregate initialization). Monster variants (minor/major/invisible/kamikaze/tiny) are separate type definitions; the `_class` bitfield enables friendly fire and faction logic. Attack shapes reference animation frame indices (frame numbers in collection sprites). The `carrying_item_type` field specifies what item a monster drops on death.
