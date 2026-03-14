# Source_Files/GameWorld/media_definitions.h

## File Purpose
Defines media type properties for the Marathon/Aleph One game engine. Specifies visual representation, damage, detonation effects, and audio for five liquid/media types (water, lava, goo, sewage, jjaro).

## Core Responsibilities
- Define the `media_definition` structure for describing media properties
- Maintain a static array of media type definitions indexed by type
- Specify visual assets (collection, shape) and rendering mode for each media
- Configure damage properties (frequency, damage type) for damaging media
- Associate detonation effects for immersion/emergence events
- Map audio effects for interaction events and ambient sounds

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `media_definition` | struct | Encodes all properties of a media type: visuals, rendering, damage, effects, and sounds |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `media_definitions` | `media_definition[NUMBER_OF_MEDIA_TYPES]` | static | Lookup table for media type properties, initialized with 5 media types |

## Key Functions / Methods
None. This file contains only structure definitions and static data initialization.

## Control Flow Notes
This is a pure data definition file. `media_definitions[]` is a lookup table accessed at runtime by game logic that needs to query properties of a specific media type (e.g., when an entity enters water, the engine consults this table for splash effects, damage, and sounds).

## External Dependencies
- **Enums / constants (defined elsewhere):**
  - `_media_water`, `_media_lava`, `_media_goo`, `_media_sewage`, `_media_jjaro` (media type identifiers)
  - `_collection_walls1`, `_collection_walls2`, etc. (sprite/texture collections)
  - `_xfer_normal` (transfer/blend mode)
  - `_damage_lava`, `_damage_goo`, `_alien_damage` (damage type identifiers)
  - `_effect_*` constants (visual effect IDs)
  - `_snd_*`, `_ambient_snd_*` constants (sound IDs)
  - `NUMBER_OF_MEDIA_TYPES`, `NUMBER_OF_MEDIA_DETONATION_TYPES`, `NUMBER_OF_MEDIA_SOUNDS` (array sizes)
  - `FIXED_ONE`, `NONE` (special values)
- **Type dependencies:**
  - `damage_definition` struct (not defined in this file)
  - `int16` (standard type)
