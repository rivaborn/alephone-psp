# Source_Files/Sound/SoundManager.cpp

## File Purpose
Core sound management system for the Aleph One game engine (Marathon). Handles sound loading, playback, channel allocation, spatial audio calculations, and parameter configuration. Acts as the primary interface between game code and the low-level Mixer.

## Core Responsibilities
- Initialize/shutdown sound system and coordinate with Mixer for audio output
- Load/unload sound definitions and manage memory-resident audio data
- Play, stop, and track active sounds across multiple prioritized channels
- Allocate channels intelligently (priority queue, volume thresholds, conflict detection)
- Calculate spatial audio properties (stereo panning, distance-based volume falloff, obstruction)
- Manage ambient and random sound sources with active culling
- Parse XML configuration for sound remapping and customization
- Support custom/replacement sound files via SoundReplacements system

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SoundManager` | class (singleton) | Main sound system controller; owns channels, sound file, and parameters |
| `SoundManager::Parameters` | nested struct | Configuration: channel count, volume, sample rate, feature flags, music volume |
| `SoundManager::Channel` | nested struct | Audio channel state: sound index, identifier, stereo/pitch variables, mixer binding |
| `Channel::Variables` | nested struct | Per-channel audio properties: volume, left/right volumes, pitch |
| `ambient_sound_data` | struct | Temporary storage during ambient sound accumulation phase |
| `XML_AmbientRandomAssignParser` | class | XML parser for ambient/random sound index assignments |
| `XML_SoundOptionsParser` | class | XML parser for custom/replacement sound file paths |
| `XML_SoundsParser` | class | XML parser for global sound event assignments |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `real_number_of_sound_definitions` | int16 | static (file) | Original sound definition count before custom additions |
| `SoundManager::m_instance` | SoundManager* | static class | Singleton instance pointer |
| `_Sound_TerminalLogon`, `_Sound_TerminalLogoff`, etc. | short | static (file) | Cached sound indices for hard-coded events (14 total) |
| `original_ambient_sound_definitions` | ambient_sound_definition* | static (file) | Backup of ambient sound table for XML reset |
| `original_random_sound_definitions` | random_sound_definition* | static (file) | Backup of random sound table for XML reset |
| `AmbientSoundAssignParser`, `RandomSoundAssignParser`, `SoundOptionsParser` | XML_*Parser | static (file) | Parser instances for XML element registration |

## Key Functions / Methods

### Initialize
- **Signature:** `void SoundManager::Initialize(const Parameters& new_parameters)`
- **Purpose:** Boot the sound system: open default sound file, register atexit handler, apply parameters.
- **Inputs:** `new_parameters` ΓÇô initial sound settings
- **Outputs/Return:** None
- **Side effects:** Sets `loaded_sounds_size = 0`, calls `OpenSoundFile()`, registers `Shutdown` as atexit handler, sets `initialized = true`
- **Calls:** `get_default_sounds_spec()`, `OpenSoundFile()`, `SetParameters()`, `SetStatus()`
- **Notes:** Must be called exactly once before any sound operations. Failure to open sound file leaves system uninitialized.

### SetStatus
- **Signature:** `void SoundManager::SetStatus(bool active)`
- **Purpose:** Activate or deactivate audio output; allocate/deallocate channels and buffers.
- **Inputs:** `active` ΓÇô whether to turn sound on or off
- **Outputs/Return:** None
- **Side effects:** On activate: calculates `total_channel_count`, `total_buffer_size`, sets up Mixer channels. On deactivate: stops all sounds, calls `Mixer::Stop()`.
- **Calls:** `StopAllSounds()`, `Mixer::instance()->Start()`, `Mixer::instance()->Stop()`, `Mixer::instance()->SoundChannelCount()`
- **Notes:** Idempotent; checks `if (active != this->active)` before acting. Mixer may report fewer channels than requested.

### PlaySound
- **Signature:** `void SoundManager::PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch)`
- **Purpose:** Play a sound with optional 3D source location and pitch modification.
- **Inputs:** `sound_index` ΓÇô which sound definition; `source` ΓÇô world location (NULL = listener-relative); `identifier` ΓÇô unique ID for tracking/stopping; `pitch` ΓÇô frequency modifier
- **Outputs/Return:** None
- **Side effects:** Loads sound, selects channel, sets channel variables, updates channel flags, buffers sound to Mixer
- **Calls:** `CalculateInitialSoundVariables()`, `LoadSound()`, `BestChannel()`, `InstantiateSoundVariables()`, `BufferSound()`, `machine_tick_count()`
- **Notes:** Validates `sound_index < number_of_sound_definitions`, `active`, `parameters.volume > 0`, `total_channel_count > 0`. If `identifier == NONE`, sound is immediately orphaned (cannot be tracked).

### DirectPlaySound
- **Signature:** `void SoundManager::DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)`
- **Purpose:** Play a sound with explicit direction and volume (no spatial calculation).
- **Inputs:** `sound_index`, `direction` (angle from listener), `volume` (explicit level), `pitch`
- **Outputs/Return:** None
- **Side effects:** Similar to `PlaySound` but bypasses spatial calculation; sets `_sound_is_local` flag
- **Calls:** `LoadSound()`, `BestChannel()`, `AngleAndVolumeToStereoVolume()`, `InstantiateSoundVariables()`, `BufferSound()`, `_sound_listener_proc()`
- **Notes:** Bug present: assignment `direction = NONE` should be comparison `direction == NONE` (line with this condition).

### LoadSound
- **Signature:** `bool SoundManager::LoadSound(short sound_index)`
- **Purpose:** Load a sound definition into memory if not already loaded; check memory budget.
- **Inputs:** `sound_index` ΓÇô which sound to load
- **Outputs/Return:** `true` if loaded successfully, `false` otherwise
- **Side effects:** Updates `loaded_sounds_size`, may evict least-used sounds, sets `definition->last_played`, clears `definition->permutations_played`
- **Calls:** `GetSoundDefinition()`, `SoundReplacements::instance()->GetSoundOptions()`, `definition->Load()`, `ReleaseLeastUsefulSound()`
- **Notes:** Respects ambient sound flag; only loads if active. Handles replacement sounds first (external files).

### BestChannel
- **Signature:** `SoundManager::Channel *SoundManager::BestChannel(short sound_index, Channel::Variables &variables)`
- **Purpose:** Select a channel for playback via priority queue; return NULL if sound cannot play.
- **Inputs:** `sound_index`, `variables` (input: desired volume/priority)
- **Outputs/Return:** Pointer to allocated Channel, or NULL
- **Side effects:** May free (mark as unused) an existing channel to make room
- **Calls:** `GetSoundDefinition()`, `local_random()`, `Mixer::instance()->ChannelBusy()`, `FreeChannel()`, `machine_tick_count()`
- **Notes:** Implements complex priority logic: rejects if sound chance fails; can abort lower-priority/lower-volume channels; prevents restart of non-restartable sounds; tracks restart delay via `machine_tick_count()`.

### Idle
- **Signature:** `void SoundManager::Idle()`
- **Purpose:** Per-frame maintenance: unlock loaded sounds, update stereo tracking, process ambient sounds.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls update functions conditionally on active/channel state
- **Calls:** `UnlockLockedSounds()`, `TrackStereoSounds()`, `CauseAmbientSoundSourceUpdate()`
- **Notes:** Should be called once per game frame.

### UpdateAmbientSoundSources
- **Signature:** `void SoundManager::UpdateAmbientSoundSources()`
- **Purpose:** Accumulate, cull, and update ambient sound channels each frame (complex state machine).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies ambient channel allocation, may call `_sound_add_ambient_sources_proc()`, calls `LoadSound()` and `BufferSound()` for ambient sounds
- **Calls:** `_sound_add_ambient_sources_proc()`, `AddOneAmbientSoundSource()`, `LoadSound()`, `BufferSound()`, `InstantiateSoundVariables()`, `FreeChannel()`
- **Notes:** Implements 5-phase algorithm: (1) accumulate sources, (2) cull zero-volume, (3) enforce max channels, (4) update existing, (5) allocate new. Ambient sounds stored at `channels[parameters.channel_count + j]`.

### CalculateSoundVariables
- **Signature:** `void SoundManager::CalculateSoundVariables(short sound_index, world_location3d *source, Channel::Variables& variables)`
- **Purpose:** Compute volume and stereo panning based on listener position, distance, and obstruction.
- **Inputs:** `sound_index` (for definition/behavior), `source` (sound origin), `variables` (output struct, reference)
- **Outputs/Return:** None (output via reference parameter)
- **Side effects:** Modifies `variables.volume`, `variables.left_volume`, `variables.right_volume`, `variables.priority`
- **Calls:** `GetSoundDefinition()`, `_sound_listener_proc()`, `distance3d()`, `distance_to_volume()`, `AngleAndVolumeToStereoVolume()`, `arctangent()`
- **Notes:** Priority = behavior index. Uses `_sound_obstructed_proc()` to determine obstruction state.

### AngleAndVolumeToStereoVolume
- **Signature:** `void SoundManager::AngleAndVolumeToStereoVolume(angle delta, short volume, short *right_volume, short *left_volume)`
- **Purpose:** Pan a mono volume into stereo left/right based on listener-relative angle.
- **Inputs:** `delta` (angle offset), `volume` (mono level)
- **Outputs/Return:** `left_volume`, `right_volume` (output pointers)
- **Side effects:** None (purely computational)
- **Calls:** None
- **Notes:** Uses 4-quadrant panning with interpolation; falls back to mono if stereo flag not set. Clamps to [min_volume, max_volume].

### Sounds_GetParser
- **Signature:** `XML_ElementParser *Sounds_GetParser()`
- **Purpose:** Factory function to build and return the root XML parser for `<sounds>` elements.
- **Inputs:** None
- **Outputs/Return:** Pointer to `SoundsParser` with children registered
- **Side effects:** Registers child parsers as children of `SoundsParser`
- **Calls:** `SoundsParser.AddChild()`
- **Notes:** Entry point for XML configuration loading.

## Control Flow Notes

**Initialization sequence:** `Initialize()` ΓåÆ `OpenSoundFile()` ΓåÆ `SetStatus(true)` ΓåÆ `Mixer::Start()` (allocate channels).

**Frame loop (per `Idle()` call):** 
- `UnlockLockedSounds()` ΓÇô mark unused sounds for eviction
- `TrackStereoSounds()` ΓÇô update stereo panning for moving sources
- `CauseAmbientSoundSourceUpdate()` ΓåÆ `UpdateAmbientSoundSources()` ΓÇô manage ambient sound playback

**Sound playback:** `PlaySound()` ΓåÆ `LoadSound()` (allocate memory if needed, evict if over budget) ΓåÆ `BestChannel()` (select or free channel) ΓåÆ `InstantiateSoundVariables()` (apply pitch/volume) ΓåÆ `BufferSound()` (send to Mixer).

**Shutdown:** `Shutdown()` ΓåÆ `SetStatus(false)` ΓåÆ `StopAllSounds()` ΓåÆ `Mixer::Stop()` ΓåÆ `CloseSoundFile()` (atexit handler).

## External Dependencies
- **Includes:** `SoundManager.h`, `ReplacementSounds.h`, `sound_definitions.h`, `Mixer.h`
- **External symbols (defined elsewhere):**
  - `SoundFile` class (manages `.snd2` file I/O)
  - `SoundReplacements` singleton (tracks custom sound overrides)
  - `Mixer` singleton (low-level audio output; SDL-based)
  - `world_location3d`, `angle` types (spatial types)
  - `_sound_listener_proc()` ΓÇô callback returning listener position/facing
  - `_sound_obstructed_proc()` ΓÇô callback checking obstruction
  - `_sound_add_ambient_sources_proc()` ΓÇô callback for ambient source enumeration
  - `distance3d()`, `arctangent()` ΓÇô math functions
  - `machine_tick_count()` ΓÇô system timer
  - `local_random()` ΓÇô RNG
  - `XML_ElementParser` (base class for parsers)
  - Constants: `MAXIMUM_SOUND_CHANNELS`, `MAXIMUM_AMBIENT_SOUND_CHANNELS`, sound behavior/volume enums
