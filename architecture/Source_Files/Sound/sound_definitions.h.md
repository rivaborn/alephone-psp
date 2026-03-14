# Source_Files/Sound/sound_definitions.h

## File Purpose
Defines sound system data structures and static sound metadata for the Aleph One/Marathon game engine. Provides audio resource format, behavior profiles, and enumeration of all in-game sounds with properties (pitch, volume, playback flags).

## Core Responsibilities
- Define sound file format (header and resource structures)
- Enumerate sound behavior profiles with distance-based volume attenuation curves
- Specify all game sounds with their properties (behavior, flags, pitch range, permutation count)
- Maintain ambient and random sound definitions
- Store and manage loaded sound pointers and metadata

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| sound_definition | struct | Audio resource metadata: sound code ID, behavior index, flags, pitch range, permutation tracking, loaded sound pointer/size |
| sound_file_header | struct | Sound resource file format: version, tag ('snd2'), source/sound counts, followed by sound definitions |
| sound_behavior_definition | struct | Volume attenuation curves for obstructed vs. unobstructed propagation |
| depth_curve_definition | struct | Distance-based volume falloff: max/min volumes with respective distances |
| ambient_sound_definition | struct | Single field: sound_index for looping ambient sounds |
| random_sound_definition | struct | Single field: sound_index for occasional one-shot sounds |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| sound_behavior_definitions[3] | sound_behavior_definition[] | static | Three profiles: quiet/normal/loud with distance curves |
| ambient_sound_definitions[28] | ambient_sound_definition[] | static | Ambient sound indices (water, machinery, alien, pfhor) |
| random_sound_definitions[4] | random_sound_definition[] | static | Random sound indices (explosions, owl, creak) |
| sound_definitions | sound_definition* | static | Dynamic runtime array of all game sounds |
| number_of_sound_definitions | long | static | Count of loaded definitions |

## Key Functions / Methods
None. This is a data definitions header.

## Control Flow Notes
No executable code. Included by sound management modules that load sound resources and play/attenuate them based on these definitions.

## External Dependencies
- FOUR_CHARS_TO_INT, MAXIMUM_SOUND_VOLUME, WORLD_ONE ΓÇö constants defined elsewhere
- Sound code enums (_snd_water, _snd_teleport_in, etc.) ΓÇö defined elsewhere
- NUMBER_OF_AMBIENT_SOUND_DEFINITIONS, NUMBER_OF_RANDOM_SOUND_DEFINITIONS ΓÇö constants defined elsewhere

## Notes
- Flags use bitmask constants for sound properties (_sound_cannot_be_restarted, _sound_is_ambient, etc.)
- Chance values encode percentages as fixed-point fractions of 32768 (e.g., _fifty_percent = 16384)
- Large commented-out legacy section shows original static sound_definitions array; now dynamically allocated at runtime
- Permutations allow multiple audio samples per sound; offsets stored relative to group_offset
- last_played tracks machine ticks to prevent rapid replaying
