# Source_Files/Sound/SoundManagerEnums.h

## File Purpose
Pure definitions header for the sound manager system. Extracted into a separate file because the main SoundManager header was becoming too large. Provides enumerations for all sound types, sound source formats, initialization flags, and obstruction conditions used throughout the audio subsystem.

## Core Responsibilities
- Define enum codes for ambient sounds (water, machinery, alien, environmental)
- Define enum codes for random/occasional sounds (drips, explosions, creaks)
- Define enum codes for triggered sounds (weapons, creatures, UI, environmental effects)
- Provide sound volume configuration constants
- Define sound source format options (8-bit vs 16-bit, 22 kHz)
- Define initialization flags for sound system configuration
- Define sound obstruction condition flags
- Define frequency/pitch adjustment options

## Key Types / Data Structures
None (enumerations only, no structs/classes/typedefs).

| Enum Category | Kind | Purpose |
|---|---|---|
| Ambient sound codes | enum | Looping background sounds: water, wind, machinery, alien noise (29 types + counter) |
| Random sound codes | enum | Occasional triggered sounds: water drips, explosions, creaks (5 types + counter) |
| Sound codes | enum | All triggered game sounds: weapons, creature vocalizations, UI, environment (300+ types + counter) |
| Sound volume | enum | Constants for volume levels (8 levels) and amplitude bits (8 bits ΓåÆ max 256) |
| Sound sources | enum | Audio format options (8-bit vs 16-bit, 22 kHz sampling) |
| Init flags | enum | Configuration bit flags for sound subsystem behavior (stereo, Doppler, memory) |
| Obstruction flags | enum | Acoustic propagation state flags (obstructed, media-obstructed, muffled) |
| Frequencies | enum | Pitch shift constants (lower/normal/higher using FIXED_ONE macro) |

## Global / File-Static State
None.

## Key Functions / Methods
None (definitions-only header).

## Control Flow Notes
Not applicable. This file is purely declarative; it provides symbolic constants consumed by SoundManager implementation and callers.

## External Dependencies
- `FIXED_ONE` macro (fixed-point arithmetic constant, defined elsewhere)
- Standard C header guards (`#ifndef`, `#endif`)

---

**Notes:**  
- Comments indicate LP (Linus Pettersson) additions and changes relative to Marathon 2 (M2).
- Commented-out entries (e.g., `_snd_nuclear_hard_death`, `_snd_unused2`) indicate removed or deprecated sounds retained for compatibility.
- The extensive sound taxonomy suggests a game with diverse weapon types, creature AI, environmental simulation, and HUD feedback.
