# Source_Files/GameWorld/weapon_definitions.h

## File Purpose
Defines the complete weapon system for the game engine, including weapon classes, firing mechanics, shell casing physics, trigger properties, and static weapon configuration data. Serves as the authoritative definition for all in-game weapons and their combat parameters.

## Core Responsibilities
- Define weapon class enums (_melee_class, _normal_class, _dual_function_class, _twofisted_pistol_class, _multipurpose_class)
- Define weapon behavior flags (automatic fire, overload capability, media-traversal, trigger sharing, etc.)
- Define weapon animation states and collections (idle, firing, reloading, charging shapes)
- Declare shell casing physics (velocity, acceleration, ejection points)
- Define trigger mechanics per weapon (ammo type, fire rate, recoil, projectile type, burst patterns)
- Store weapon ordering and configurations as static/const arrays
- Provide serialization/deserialization functions for weapon definitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| shell_casing_definition | struct | Defines physics for ejected shell casings (position offset, velocity, acceleration) |
| trigger_definition | struct | Single trigger/fire mode: ammo type, fire rate, recoil, sounds, projectile type, burst count |
| weapon_definition | struct | Complete weapon: class, flags, animations, timing windows, two trigger configurations |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| shell_casing_definitions | shell_casing_definition[5] | static | Physics data for assault rifle, pistol (3 variants), and SMG casings |
| weapon_ordering_array | int16[] | static | Display/selection order: fist, pistol, plasma, shotgun, assault rifle, SMG, flamethrower, missile, alien shotgun, ball |
| weapon_definitions | weapon_definition[10] | static | Mutable weapon array; initialized from original_weapon_definitions |
| original_weapon_definitions | weapon_definition[10] const | const global | Default weapon configuration (fist, magnum, fusion pistol, assault rifle, rocket launcher, flamethrower, alien shotgun, shotgun, ball, SMG) |
| NUMBER_OF_WEAPONS | int (macro) | compile-time | = 10 |

## Key Functions / Methods

### unpack_weapon_definition
- Signature: `uint8 *unpack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- Purpose: Deserialize weapon definitions from a binary stream (e.g., from saved game or data file)
- Inputs: byte stream pointer, destination array, count
- Outputs/Return: pointer to next position in stream
- Side effects: populates weapon_definition array
- Calls: not implemented in this file
- Notes: declaration only; implementation in another file

### pack_weapon_definition
- Signature: `uint8 *pack_weapon_definition(uint8 *Stream, weapon_definition *Objects, size_t Count)`
- Purpose: Serialize weapon definitions to a binary stream for storage/transmission
- Inputs: byte stream pointer, source array, count
- Outputs/Return: pointer to next position in stream
- Side effects: writes to stream
- Calls: not implemented in this file
- Notes: declaration only; implementation in another file

## Control Flow Notes
This file is purely **declarative data**. It does not participate in frame/update/render loops. Weapons are initialized at engine startup by reading these definitions and populating the weapon_definition array, then accessed during gameplay for fire rate, animation, and projectile calculations.

## External Dependencies
- Type definitions: `int16`, `_fixed` (fixed-point), `world_distance`, `uint8` ΓÇö defined elsewhere
- Constants: `FIXED_ONE`, `FIXED_ONE_HALF`, `TICKS_PER_SECOND`, `WORLD_ONE_FOURTH`, `NORMAL_WEAPON_DZ` ΓÇö defined elsewhere
- Enums (referenced as item types, sounds, projectiles, etc.): `_i_knife`, `_snd_magnum_firing`, `_projectile_fist`, etc. ΓÇö defined elsewhere
- Collection indices: `_weapon_in_hand_collection`, `_collection_player` ΓÇö defined elsewhere
- `NUMBER_OF_TRIGGERS` macro ΓÇö defined elsewhere (likely = 2 for primary/secondary)
