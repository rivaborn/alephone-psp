# Source_Files/Sound/SoundManager.h

## File Purpose
Central audio subsystem for the Aleph One game engine. Manages sound initialization, loading, playback with 3D spatial audio, channel allocation, volume control, and ambient sound sources. Implements a singleton audio manager coordinating between sound files, mixer channels, and the game world.

## Core Responsibilities
- Initialize and configure the audio subsystem with parameters (sample rate, channels, flags)
- Load and unload sound definitions from sound resource files
- Play sounds with 3D world-space positioning, direction, volume, and pitch control
- Allocate and manage audio mixer channels; select best channel for new sounds
- Calculate spatial audio: stereo panning, pitch modulation, attenuation based on listener distance
- Track and update ambient sound sources in the game world
- Handle dynamic sound memory management (loading/unloading/disposal)
- Provide pause/resume functionality via RAII helper
- Query sound state (is sound playing, number of definitions, netmic volume adjustment)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `Parameters` | struct | Configuration: channel count, volume levels, sample rate, buffer size, flags (stereo, dynamic tracking, doppler shift, ambient sounds, 16-bit mode) |
| `Channel` | struct | Active playback channel: sound index, identifier, volume variables (left/right), pitch, priority, dynamic source location, mixer channel binding, callback counter |
| `Pause` | class | RAII scope guard: pauses sounds on construction, resumes on destruction |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `SoundManager*` | static | Singleton instance |
| `initialized` | `bool` | member | Whether subsystem is fully initialized |
| `active` | `bool` | member | Whether audio playback is enabled |
| `channels` | `std::vector<Channel>` | member | All mixer channels (max 32 + 4 ambient) |
| `sound_file` | `SoundFile` | member | Loaded sound resource file |
| `total_channel_count` | `short` | member | Current total channels in use |
| `total_buffer_size` | `long` | member | Current allocated buffer size |
| `loaded_sounds_size` | `long` | member | Size of currently loaded sound data |
| `MAXIMUM_SOUND_CHANNELS` | const int | static | 32 channels max |
| `MAXIMUM_AMBIENT_SOUND_CHANNELS` | const int | static | 4 ambient channels max |
| `MAXIMUM_SOUND_VOLUME` | const int | static | 256 (8-bit) |
| `NUMBER_OF_SOUND_VOLUME_LEVELS` | const int | static | 8 levels |

## Key Functions / Methods

### instance()
- Signature: `static inline SoundManager* instance()`
- Purpose: Singleton accessor; lazy-initializes instance on first call
- Inputs: None
- Outputs/Return: Pointer to singleton `SoundManager` instance
- Side effects: Creates instance if not already allocated
- Notes: Not thread-safe; assumes single-threaded engine

### Initialize(const Parameters&)
- Signature: `void Initialize(const Parameters&)`
- Purpose: Set up audio subsystem with given configuration
- Inputs: Parameters struct (channels, volumes, sample rate, flags)
- Outputs/Return: None (void)
- Side effects: Allocates mixer channels, configures audio device, sets `initialized` flag
- Calls: Internal setup routines, likely SDL audio initialization
- Notes: Must be called before any sound playback

### Shutdown()
- Signature: `void Shutdown()`
- Purpose: Gracefully shut down the audio subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Stops all sounds, deallocates channels and buffers, closes sound file
- Calls: `StopAllSounds()`, `CloseSoundFile()`, memory cleanup

### OpenSoundFile(FileSpecifier& File) / CloseSoundFile()
- Signature: `bool OpenSoundFile(FileSpecifier& File)` / `void CloseSoundFile()`
- Purpose: Load/unload the main sound resource file
- Inputs: File path specifier or none
- Outputs/Return: bool success, void
- Side effects: Opens/closes sound file, validates version/structure
- Calls: SoundFile::Open(), FileHandler I/O

### PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch)
- Signature: `void PlaySound(short sound_index, world_location3d *source, short identifier, _fixed pitch = _normal_frequency)`
- Purpose: Main entry point for 3D spatial sound playback
- Inputs: Sound definition index, 3D world location of source (NULL for immobile), unique sound ID, pitch modifier
- Outputs/Return: None
- Side effects: Allocates mixer channel, loads sound data, calculates spatial audio variables, begins playback
- Calls: `BestChannel()`, `BufferSound()`, `CalculateInitialSoundVariables()`, `InstantiateSoundVariables()`
- Notes: `identifier` used to stop specific sound instances; NULL source = non-positional sound

### PlayLocalSound(short sound_index, _fixed pitch)
- Signature: `void PlayLocalSound(short sound_index, _fixed pitch = _normal_frequency)`
- Purpose: Play non-positional sound (e.g., UI feedback, player-centric audio)
- Inputs: Sound index, pitch
- Outputs/Return: None
- Side effects: Delegates to `PlaySound()` with NULL source location
- Notes: Convenience wrapper; no attenuation or panning

### DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)
- Signature: `void DirectPlaySound(short sound_index, angle direction, short volume, _fixed pitch)`
- Purpose: Play sound with explicit direction and volume (bypassing 3D calculation)
- Inputs: Sound index, direction angle, volume level, pitch
- Outputs/Return: None
- Side effects: Allocates channel, plays with given parameters without spatial calculation
- Notes: Used for directional non-positional sounds (e.g., compass direction audio)

### Idle()
- Signature: `void Idle()`
- Purpose: Per-frame update of active sounds; called every game tick
- Inputs: None
- Outputs/Return: None
- Side effects: Updates sound positions, applies doppler shift, updates ambient sources, tracks stereo
- Calls: `TrackStereoSounds()`, `UpdateAmbientSoundSources()`, spatial calculations
- Notes: Must be called regularly for 3D audio and ambient sound updates

### LoadSound(short) / LoadSounds(short*, short)
- Signature: `bool LoadSound(short sound)` / `void LoadSounds(short *sounds, short count)`
- Purpose: Pre-load sound data into memory
- Inputs: Sound index or array of indices
- Outputs/Return: bool success or void
- Side effects: Loads sound definition and audio data from SoundFile; updates `loaded_sounds_size`
- Calls: `GetSoundDefinition()`, SoundFile::Load()

### StopSound(short identifier, short sound_index) / StopAllSounds()
- Signature: `void StopSound(short identifier, short sound_index)` / `void StopAllSounds()`
- Purpose: Stop playback of specific sound or all sounds
- Inputs: Sound identifier and/or index (NONE = wildcard)
- Outputs/Return: None
- Side effects: Releases mixer channels, clears channel variables
- Calls: `FreeChannel()`

### CalculateSoundVariables(short sound_index, world_location3d *source, Channel::Variables& variables)
- Signature: `void CalculateSoundVariables(short sound_index, world_location3d *source, Channel::Variables& variables)`
- Purpose: Compute audio variables (volume, stereo panning, pitch) based on 3D source position relative to listener
- Inputs: Sound index, source location, output variables struct
- Outputs/Return: None (modifies variables in-place)
- Side effects: Calls `_sound_listener_proc()` to get listener position; calls `_sound_obstructed_proc()` for obstruction testing
- Calls: Trig functions, distance calculations from `world.h`, obstruction queries
- Notes: Core 3D audio algorithm; computes distance attenuation, angle-to-stereo mapping, pitch shift (if doppler enabled)

### BestChannel(short sound_index, Channel::Variables& variables)
- Signature: `Channel* BestChannel(short sound_index, Channel::Variables& variables)`
- Purpose: Select or allocate mixer channel for new sound; may evict lower-priority sound
- Inputs: Sound index (for priority), audio variables
- Outputs/Return: Pointer to allocated channel or NULL
- Side effects: May call `ReleaseLeastUsefulSound()` if all channels full
- Calls: Channel search/allocation logic
- Notes: Implements channel priority system; considers sound priority, volume, playback duration

### CauseAmbientSoundSourceUpdate()
- Signature: `void CauseAmbientSoundSourceUpdate()`
- Purpose: Trigger refresh of ambient sound source list
- Inputs: None
- Outputs/Return: None
- Side effects: Calls external callback `_sound_add_ambient_sources_proc()` to enumerate ambient sources
- Calls: `_sound_add_ambient_sources_proc()`, `AddOneAmbientSoundSource()`

### Pause (RAII class)
- Constructor: `Pause()` ΓÇö calls `SetStatus(false)`
- Destructor: `~Pause()` ΓÇö calls `SetStatus(true)`
- Purpose: Automatically pause/resume sounds within a scope
- Usage: `{ SoundManager::Pause p; /* sounds paused */ }` ΓÇö sounds resume on `}`

## Control Flow Notes

**Initialization phase:**
1. Engine calls `SoundManager::instance()` ΓåÆ singleton created
2. Engine calls `Initialize(Parameters)` ΓåÆ audio device configured, channels allocated
3. Engine calls `OpenSoundFile(FileSpecifier)` ΓåÆ sound definitions loaded

**Per-frame audio loop:**
1. Game calls `SoundManager::Idle()` each frame
2. `Idle()` updates all active channels:
   - Calls `TrackStereoSounds()` to update stereo balance for moving sources
   - Calls `UpdateAmbientSoundSources()` to refresh ambient sound positions
3. Spatial audio applied via `CalculateSoundVariables()` using current listener position

**Sound playback (on PlaySound):**
1. `PlaySound()` calls `BestChannel()` to allocate a mixer channel
2. Calls `CalculateInitialSoundVariables()` to compute initial volume/pitch/stereo
3. Calls `BufferSound()` to load audio data to mixer
4. Calls `InstantiateSoundVariables()` to apply variables to channel
5. Mixer then plays the buffered sound

**Shutdown phase:**
1. Engine calls `Shutdown()`
2. Stops all sounds, deallocates channels, closes sound file

## External Dependencies

**Included headers:**
- `cseries.h` ΓÇö Common utilities, data types
- `FileHandler.h` ΓÇö File I/O (`FileSpecifier`, `OpenedFile`)
- `SoundFile.h` ΓÇö Sound resource loading (`SoundFile`, `SoundDefinition`)
- `world.h` ΓÇö World geometry (`world_location3d`, `angle`, `_fixed`, `world_distance`)
- `XML_ElementParser.h` ΓÇö XML config parsing
- `SoundManagerEnums.h` ΓÇö Sound ID enumerations, flags, constants

**External functions (defined elsewhere):**
- `world_location3d *_sound_listener_proc(void)` ΓÇö Gets current listener position/orientation
- `uint16 _sound_obstructed_proc(world_location3d *source)` ΓÇö Tests obstruction between source and listener
- `void _sound_add_ambient_sources_proc(...)` ΓÇö Callback to enumerate ambient sound sources
- `short Sound_TerminalLogon()`, `Sound_Breathing()`, etc. ΓÇö Sound ID accessors
- `XML_ElementParser *Sounds_GetParser()` ΓÇö XML config parser factory

**Uses (from world.h):**
- `world_location3d` (3D point + polygon + orientation)
- `angle`, `_fixed`, `world_distance` types
- Trigonometry macros and distance functions
