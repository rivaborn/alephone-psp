# Source_Files/GameWorld/projectile_definitions.h

## File Purpose
Defines the properties and behavior of all projectile types in the game. Serves as the central data repository for projectile metadata, including damage, visual effects, physics parameters, and behavioral flags. Initialized at game startup and referenced by the projectile system during runtime.

## Core Responsibilities
- Define projectile behavior flags (guidance, gravity, media penetration, melee properties, etc.)
- Declare the `projectile_definition` structure that encapsulates all projectile properties
- Initialize static and const arrays of projectile definitions for ~40+ projectile types
- Provide serialization interfaces for saving/loading projectile data
- Support both player and alien weapon projectiles with distinct behavior profiles

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| (projectile flags enum) | enum (anonymous) | Bitflags controlling projectile behavior: guidance, gravity, media handling, transparency, melee properties, detonation rules |
| `projectile_definition` | struct | Encapsulates all properties of a projectile type: visual/audio assets, damage, physics (speed/range), effects, and behavioral flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `projectile_definitions` | `projectile_definition[]` | static (module) | Mutable working copy of all projectile definitions; can be modified at runtime |
| `original_projectile_definitions` | `projectile_definition[]` | const (module) | Immutable baseline definitions; used for reset or comparison |

## Key Functions / Methods

### unpack_projectile_definition
- Signature: `uint8 *unpack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- Purpose: Deserialize projectile data from a byte stream (e.g., from saved game or network message)
- Inputs: Byte stream pointer, target projectile array, count of projectiles to unpack
- Outputs/Return: Updated stream pointer (position after read)
- Side effects: Modifies `Objects` array in-place; advances stream pointer
- Calls: Not visible in this file (defined elsewhere)
- Notes: Inverse of `pack_projectile_definition`; used for game state persistence

### pack_projectile_definition
- Signature: `uint8 *pack_projectile_definition(uint8 *Stream, projectile_definition *Objects, size_t Count)`
- Purpose: Serialize projectile data to a byte stream
- Inputs: Byte stream pointer, source projectile array, count of projectiles to pack
- Outputs/Return: Updated stream pointer (position after write)
- Side effects: Writes to stream; advances pointer
- Calls: Not visible in this file (defined elsewhere)
- Notes: Inverse of `unpack_projectile_definition`; used for game state persistence

## Control Flow Notes
This is a data-definition file with no control flow. It initializes static data at program startup. The `projectile_definitions` array is populated from `original_projectile_definitions` at game init (mechanism not visible here), then referenced by the projectile simulation system during game updates. Serialization functions support save/load mechanics.

## External Dependencies
- **`damage_definition`** ΓÇö struct (defined elsewhere); embedded in each projectile definition
- **Effect/sound enums** ΓÇö `_effect_*`, `_snd_*` constants (defined elsewhere); index into game asset tables
- **`world_distance`** ΓÇö typedef for spatial measurements (likely fixed-point or scaled integer)
- **`_fixed`** ΓÇö typedef for sound pitch values (likely fixed-point number)
- **`BUILD_COLLECTION()`** ΓÇö macro (defined elsewhere); constructs collection indices
- **`NUMBER_OF_PROJECTILE_TYPES`** ΓÇö constant (defined elsewhere); array dimension
- **Damage type enums** ΓÇö `_damage_explosion`, `_damage_projectile`, `_alien_damage`, etc. (defined elsewhere)

---

**Notes:** The file is licensed under GPL and references the Marathon/Aleph One project (Bungie Studios). The projectile system supports ~40 types ranging from player weapons (rockets, grenades, SMG bullets) to alien weapons (compiler bolts, fusion bolts, hunter projectiles) and special effects (fist, ball, energy drain). Many projectiles support complex behaviors: media penetration, guided targeting, detonation cascades, and gravity variations.
