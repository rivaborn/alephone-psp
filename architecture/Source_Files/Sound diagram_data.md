# Source_Files/Sound/BasicIFFDecoder.cpp
## File Purpose
Decodes uncompressed WAV and AIFF audio files for playback. Extracts audio metadata (sample rate, bit depth, channels, endianness) from file headers and provides frame-by-frame audio data reading with playback controls (rewind, seek, done detection).

## Core Responsibilities
- Parse AIFF and WAV file headers to extract audio format information
- Detect and handle big-endian (AIFF) and little-endian (WAV) byte ordering
- Extract audio metadata: sample rate, channel count, bit depth, frame size
- Read and return raw audio frames to decoder consumer
- Manage file position during playback (current position, data offset, total length)
- Support playback control: rewind to start, detect end-of-file, close file handle

## External Dependencies
- **SDL_endian.h**: `SDL_ReadBE32`, `SDL_ReadLE32`, `SDL_ReadBE16`, `SDL_ReadLE16`, `SDL_RWops`, `SDL_RWtell`, `SDL_RWseek` (byte-order-aware binary I/O)
- **Decoder** (base class, defined elsewhere): Pure interface for audio decoders
- **FileSpecifier** (defined elsewhere): Abstraction over file I/O with position tracking
- **std::vector** (included but unused in this file)

# Source_Files/Sound/BasicIFFDecoder.h
## File Purpose
Decoder class for uncompressed AIFF and WAV audio files. Inherits from the `Decoder` abstract base class and implements frame-based streaming decoding with format metadata (bit depth, channels, sample rate, endianness).

## Core Responsibilities
- Open and validate uncompressed AIFF/WAV files
- Decode audio frames into a provided buffer
- Track playback position and provide frame count
- Expose audio format properties (mono/stereo, 8/16-bit, signed/unsigned, sample rate)
- Manage file handle and buffer offset during streaming

## External Dependencies
- `Decoder.h` ΓÇô parent class `Decoder : public StreamDecoder`
- `FileHandler.h` ΓÇô `FileSpecifier`, `OpenedFile` (defined elsewhere)
- `cseries.h` ΓÇô basic types (`uint8`, `int32`, `float`; defined elsewhere)

# Source_Files/Sound/Decoder.cpp
## File Purpose
Factory methods for instantiating audio format decoders based on available compile-time features and file format detection. Attempts decoders in priority order, returning the first one that successfully opens the given file.

## Core Responsibilities
- Static factory for StreamDecoder instances (supports SNDFILE, VORBIS, MAD, and BasicIFF fallback)
- Static factory for Decoder instances (frame-aware decoders; BasicIFF fallback)
- Format auto-detection via sequential decoder instantiation and Open() attempts
- Ownership transfer via auto_ptr::release() to caller
- Conditional compilation to select available audio codec support

## External Dependencies
- **Includes:** Decoder.h, BasicIFFDecoder.h, MADDecoder.h, SndfileDecoder.h, VorbisDecoder.h, memory (std::auto_ptr)
- **Defined elsewhere:** FileSpecifier (file abstraction), all Decoder subclasses (BasicIFFDecoder, MADDecoder, SndfileDecoder, VorbisDecoder)
- **Build-time toggles:** HAVE_SNDFILE, HAVE_VORBISFILE, HAVE_MAD

# Source_Files/Sound/Decoder.h
## File Purpose
Defines abstract base classes for audio decoding with factory pattern support. `StreamDecoder` provides streaming decode operations and audio format queries; `Decoder` extends it to add frame-count capability for full-file operations.

## Core Responsibilities
- Provide abstract interface for opening, decoding, and closing audio streams
- Expose audio format metadata (bit depth, sample rate, channel count, endianness, signedness)
- Support factory pattern for instantiating appropriate decoder implementations based on file type
- Separate streaming decoders from full-file decoders with frame information
- Allow repeated decoding and rewinding of audio data

## External Dependencies
- `cseries.h` ΓÇô engine type definitions (`int32`, `uint8`, `bool`)
- `FileHandler.h` ΓÇô `FileSpecifier` class for file abstraction
- Concrete decoder implementations not defined in this header (e.g., WAV, Ogg, MIDI decoders)

# Source_Files/Sound/MADDecoder.cpp
## File Purpose
Implements MP3 audio decoding for Aleph One using libmad. Manages the libmad stream decoder, frame-by-frame MP3 decompression, and conversion of fixed-point audio samples to PCM 16-bit signed integers with endianness handling.

## Core Responsibilities
- MP3 file decoding via libmad frame decoder
- Input stream buffer management and refilling
- Conversion of libmad fixed-point samples to 16-bit PCM with endianness adaptation
- Audio format detection (sample rate, channel count)
- File position management (seeking, rewinding)
- Graceful end-of-file and error handling with guard bytes for libmad

## External Dependencies
- **Includes:** `<mad.h>` (libmad MP3 decoder library)
- **Inherits from:** `StreamDecoder` (defined elsewhere, likely abstract interface)
- **Uses:** `FileSpecifier` (file I/O abstraction, defined elsewhere)
- **Type macros:** `uint8`, `int16`, `int32`, `MAD_F_ONE`, `MAD_F_FRACBITS`, `SHRT_MAX` (platform/config defines)
- **Conditional compilation:** `#ifdef HAVE_MAD`, `#ifdef ALEPHONE_LITTLE_ENDIAN`

# Source_Files/Sound/MADDecoder.h
## File Purpose
Declares the `MADDecoder` class, an MP3 audio decoder that wraps the libmad library. Enables the engine to decode and stream MP3 files, providing access to audio samples with format metadata (channels, bit depth, sample rate).

## Core Responsibilities
- Opens and manages MP3 file streams via libmad
- Decodes MP3 frames and produces raw PCM audio samples
- Provides format queries (stereo, 16-bit, signed, sample rate, endianness)
- Maintains audio decoding state (current frame, synthesis buffers, input stream)
- Handles file I/O buffering and rewind/close operations

## External Dependencies
- **libmad** (`<mad.h>`) ΓÇô MP3 frame parsing, synthesis
- **cseries.h** ΓÇô project type definitions and utilities
- **Decoder.h** ΓÇô `StreamDecoder` base class interface and factory
- **FileHandler.h** (via cseries) ΓÇô `FileSpecifier` and `OpenedFile` types
- **Conditional compilation:** `HAVE_MAD`, `ALEPHONE_LITTLE_ENDIAN`

# Source_Files/Sound/Mixer.cpp
## File Purpose
Implements the core audio mixing engine for the game, integrating with SDL for cross-platform audio output. Manages multiple concurrent sound channels (effects, music, network audio, resources), handles format conversions and resampling, and produces the final audio stream mixed from all active sources.

## Core Responsibilities
- Initialize and manage SDL audio subsystem with configurable sample rate, bit depth, and channel count
- Maintain a vector of audio channels with independent playback state and format metadata
- Queue and buffer sound data to channels with pitch/rate control and looping support
- Execute real-time audio mixing in SDL callback context, interpolating samples and applying per-channel volume
- Handle music streaming via `Music` subsystem with buffer refills during playback
- Manage network microphone audio dequeue and playback with dynamic buffer lifecycle
- Support format-agnostic mixing (8/16-bit, mono/stereo, signed/unsigned, little/big-endian)
- Apply master volume and mute networked audio during local transmission

## External Dependencies
- **SDL:** `SDL_OpenAudio()`, `SDL_CloseAudio()`, `SDL_PauseAudio()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`, `SDL_RWFromMem()`, `SDL_RWops`, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SDL_RWseek()`, `SDL_RWclose()`, `SDL_SwapLE16()`, `SDL_SwapBE16()`, `SDL_AudioSpec`
- **Music subsystem:** `Music::instance()->FillBuffer()`, `Music::instance()->InterruptFillBuffer()`
- **SoundManager:** `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `SoundManager::instance()->IncrementChannelCallbackCount()`
- **Network audio:** `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`, `kNetworkAudioIsStereo`, `kNetworkAudioIs16Bit`, `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame`, `kNetworkAudioIsSigned8Bit`
- **interface.h:** `strERRORS`, `badSoundChannels`, `alert_user()`, `infoError`
- **Global state:** `game_is_networked`, `local_player_index`, `dynamic_world->speaking_player_index`
- **SoundHeader class:** Loaded from resources; provides format and sample data

# Source_Files/Sound/Mixer.h
## File Purpose
Implements the audio mixer for the Aleph One game engine, managing real-time mixing of multiple audio channels (sound effects, music, network audio) into a single SDL output stream. Handles format conversion, pitch shifting, and volume control for playback.

## Core Responsibilities
- Singleton mixer that manages SDL audio initialization and the audio callback
- Maintains multiple playback channels with format metadata and sample state
- Performs real-time sample mixing with linear interpolation for pitch shifting
- Handles diverse audio formats (8/16-bit, mono/stereo, signed/unsigned, endianness)
- Applies per-channel and master volume control with networking-aware muting
- Interfaces with Music and network audio systems to queue decoded audio data
- Manages sound resource playback on dedicated channels

## External Dependencies

- **SDL_endian.h** ΓÇô Endianness conversion macros (SDL_SwapLE16, SDL_SwapBE16)
- **cseries.h** ΓÇô Common type definitions (_fixed, uint8, int16, etc.)
- **network_speaker_sdl.h** ΓÇô NetworkSpeakerSoundBufferDescriptor, dequeue_network_speaker_data(), release_network_speaker_buffer()
- **network_audio_shared.h** ΓÇô Network audio format constants
- **map.h** ΓÇô External dynamic_world (accessed for speaking_player_index during muting logic)
- **Music.h** ΓÇô Music::instance(), FillBuffer(), InterruptFillBuffer() 
- **SoundManager.h** ΓÇô SoundManager::instance(), GetNetmicVolumeAdjustment(), IncrementChannelCallbackCount()

# Source_Files/Sound/Music.cpp
## File Purpose
Implements singleton music playback manager for intro and level music in the Aleph One game engine. Handles decoder setup, audio buffering, fade-out effects, and level music playlists, delegating actual mixing to the Mixer component.

## Core Responsibilities
- Load and manage music files via StreamDecoder and Mixer integration
- Maintain intro music and level music playback state
- Implement fade-out with time-based volume interpolation
- Manage level music playlists with sequential or random song selection
- Fill audio buffers with decoded frames during playback
- Handle platform-specific (macOS) interrupt-safe buffer updates
- Coordinate initialization, volume checking, and playback state with SoundManager

## External Dependencies
- **Notable includes:** Music.h, Mixer.h, XML_LevelScript.h
- **SDL:** SDL_GetTicks() (timing), SDL_RWops* (unused in this file)
- **StreamDecoder:** Decoder::Get(), decoderΓåÆDecode()/Rewind()/IsSixteenBit/IsStereo/IsSigned/BytesPerFrame/Rate/IsLittleEndian
- **FileSpecifier:** File path wrapper; equality comparison
- **SoundManager:** instance(), IsInitialized(), IsActive(), parameters.music, GetNetmicVolumeAdjustment()
- **Mixer:** instance(), MusicPlaying(), StartMusicChannel(), UpdateMusicChannel(), SetMusicChannelVolume(), StopMusicChannel(), obtained.freq
- **GM_Random:** randomizer.KISS(), randomizer.SetTable()
- **XML_LevelScript:** Not directly called from this file (included but unused)

# Source_Files/Sound/Music.h
## File Purpose
Singleton manager for intro and level music playback in the Aleph One game engine. Handles audio stream decoding, buffer management, fade effects, and platform-specific audio I/O. Supports per-level music playlists with optional randomization.

## Core Responsibilities
- Music playback lifecycle (open, play, pause, stop, close, restart, rewind)
- Fade in/out effects with configurable durations
- Audio stream decoding and buffer filling for real-time playback
- Level music playlist management with random ordering
- Volume synchronization with SoundManager
- Platform-specific buffer handling (separate logic for macOS)
- Stream format detection (bit depth, channels, endianness, sample rate)

## External Dependencies
- **cseries.h**: Base types (`uint32`, `uint8`, `int16`, `_fixed`), macros, SDL includes
- **Decoder.h**: `StreamDecoder` abstract interface
- **FileHandler.h**: `FileSpecifier` file abstraction
- **Random.h**: `GM_Random` random number generator
- **SoundManager.h**: Access to global music volume via `SoundManager::instance()->parameters.music`
- **SDL.h** (via cseries): `SDL_RWops` file I/O

# Source_Files/Sound/ReplacementSounds.cpp
## File Purpose
Implements a singleton manager for external sound replacements in the Aleph One game engine. Loads audio files from disk, decodes them, and provides hash-table-based lookup for sound options indexed by audio type (Index) and variant (Slot).

## Core Responsibilities
- Load and decode external audio files via `Decoder` abstraction
- Manage a collection of sound options with efficient lookup by (Index, Slot) pair
- Implement a hash table with linear-search fallback for sound option retrieval
- Add or update sound options, avoiding duplicates by (Index, Slot)
- Maintain singleton instance of the replacement sound manager

## External Dependencies
- **Decoder.h** ΓÇô `Decoder::Get()` factory; query methods (`Frames()`, `BytesPerFrame()`, `Decode()`, `IsSixteenBit()`, etc.)
- **SoundFile.h** ΓÇô `SoundHeader` base class (`Load()`, `Clear()` methods)
- **FileHandler.h** ΓÇô `FileSpecifier` type
- **cseries.h** ΓÇô Low-level types (`int32`, `uint8`, `NONE`, `FIXED_ONE`)
- **C++ std** ΓÇô `std::vector`, `std::auto_ptr` (deprecated C++98 idiom; should be `std::unique_ptr` in modern C++)

# Source_Files/Sound/ReplacementSounds.h
## File Purpose
Provides a singleton-managed system for loading and retrieving external sound files as replacements for built-in game sounds. Implements hash-based lookup for efficient runtime sound retrieval during gameplay based on sound index and permutation slot.

## Core Responsibilities
- Define `ExternalSoundHeader` to load external sound files from disk
- Manage a registry of sound replacements indexed by sound ID and permutation slot
- Provide O(1) hash-table lookup for real-time sound retrieval
- Singleton access pattern for global sound replacement management
- Support reset/clearing of all replacements

## External Dependencies
- `<string>` ΓÇö for file paths
- `SoundFile.h` ΓÇö provides `SoundHeader` base class and `FileSpecifier` type
- Aleph One engine framework (license headers, file I/O abstractions)


# Source_Files/Sound/SndfileDecoder.cpp
## File Purpose
Implements a sound decoder that wraps libsndfile, enabling the engine to read and decode audio samples from various sound file formats (WAV, FLAC, etc.). Provides a simple streaming interface for decoding audio data in chunks and seeking within files.

## Core Responsibilities
- Open sound files and validate format metadata via libsndfile
- Decode audio samples into a buffer for playback
- Rewind/seek to the beginning of a sound file
- Manage lifetime of libsndfile file handles (open/close)
- Safely handle resource cleanup in constructor and destructor

## External Dependencies
- `sndfile.h` ΓÇô libsndfile C API (file I/O, format detection, PCM decoding)
- `Decoder.h` ΓÇô base class defining the audio decoder interface
- `FileSpecifier` ΓÇô engine's file abstraction (GetPath method used)

# Source_Files/Sound/SndfileDecoder.h
## File Purpose
Declares `SndfileDecoder`, a concrete decoder for audio files using the libsndfile library. Inherits from the abstract `Decoder` base class to provide streaming PCM decoding for the game engine's audio system. Only compiled when libsndfile support is available (guarded by `#ifdef HAVE_SNDFILE`).

## Core Responsibilities
- Decode audio files (WAV, FLAC, OGG Vorbis, etc.) supported by libsndfile into raw PCM buffers
- Manage the lifecycle of a libsndfile decoder handle (`SNDFILE*`)
- Report audio format metadata (bit depth, channels, sample rate, endianness, total frame count)
- Support sequential decoding and stream rewinding
- Maintain consistency with the abstract decoder interface for polymorphic usage

## External Dependencies
- **`Decoder.h`** ΓÇö abstract base class `Decoder` and `StreamDecoder`
- **`sndfile.h`** ΓÇö external libsndfile library (provides `SNDFILE`, `SF_INFO`, and decode functions)
- **`FileHandler.h`** ΓÇö defines `FileSpecifier` (via Decoder.h)
- **`cseries.h`** ΓÇö defines standard types like `uint8`, `int32` (via Decoder.h)

# Source_Files/Sound/song_definitions.h
## File Purpose
Defines C structures and constants for organizing game music tracks into segments. Part of the Aleph One engine's audio subsystem. Provides a data-driven format for specifying song metadata including introduction, chorus, and trailer segments with looping behavior.

## Core Responsibilities
- Define `sound_snippet` struct for specifying audio segment boundaries (start/end offsets)
- Define `song_definition` struct to organize a complete song's metadata
- Provide flag constants for song behavior (`_song_automatically_loops`)
- Define a macro (`RANDOM_COUNT`) for representing randomized chorus repetition counts
- Declare a global `songs[]` array as the engine's song registry

## External Dependencies
- **External symbols used**: `MACHINE_TICKS_PER_SECOND` (macro, likely from platform/timing header)
- **Type assumptions**: `int16`, `int32` (platform-dependent integer types, likely from a common header)
- **Standard**: C89/C99 struct definitions with fixed-width integer types


# Source_Files/Sound/sound_definitions.h
## File Purpose
Defines sound system data structures and static sound metadata for the Aleph One/Marathon game engine. Provides audio resource format, behavior profiles, and enumeration of all in-game sounds with properties (pitch, volume, playback flags).

## Core Responsibilities
- Define sound file format (header and resource structures)
- Enumerate sound behavior profiles with distance-based volume attenuation curves
- Specify all game sounds with their properties (behavior, flags, pitch range, permutation count)
- Maintain ambient and random sound definitions
- Store and manage loaded sound pointers and metadata

## External Dependencies
- FOUR_CHARS_TO_INT, MAXIMUM_SOUND_VOLUME, WORLD_ONE ΓÇö constants defined elsewhere
- Sound code enums (_snd_water, _snd_teleport_in, etc.) ΓÇö defined elsewhere
- NUMBER_OF_AMBIENT_SOUND_DEFINITIONS, NUMBER_OF_RANDOM_SOUND_DEFINITIONS ΓÇö constants defined elsewhere


# Source_Files/Sound/SoundFile.cpp
## File Purpose
Implements sound file loading and management for the Aleph One game engine. Handles parsing System 7 sound formats, loading sound definitions and permutations, and supports dynamic custom sound addition via external decoders.

## Core Responsibilities
- Parse and unpack System 7 sound headers (standard 22-byte and extended 64-byte formats)
- Load sound sample data from files into memory
- Manage hierarchical sound definitions (sources ΓåÆ sounds ΓåÆ permutations)
- Support dynamic custom sound definitions and external format loading
- Validate sound file structure and metadata (magic tag, version, sample rates, loop points)

## External Dependencies

- **AStream.h** ΓÇö `AIStreamBE` (big-endian binary stream reader)
- **FileHandler.h** ΓÇö `OpenedFile`, `FileSpecifier` (file I/O abstraction)
- **Decoder.h** ΓÇö `Decoder` (external sound format decoder; supports multiple codecs)
- **Logging.h** ΓÇö `logWarning3` (warning logging)
- **csmisc.h** ΓÇö `machine_tick_count()` (platform tick counter), `FIXED_ONE` (fixed-point unity)
- **Standard library** ΓÇö `<vector>`, `<memory>` (auto_ptr), `<assert.h>`

**Defined elsewhere:** `FOUR_CHARS_TO_INT` macro, `MAXIMUM_PERMUTATIONS_PER_SOUND` constant.

# Source_Files/Sound/SoundFile.h
## File Purpose
Defines sound resource management classes for loading, storing, and accessing sound data in System 7 format. Part of the Aleph One game engine's audio subsystem, handling both built-in and custom sound definitions with pitch variation and looping support.

## Core Responsibilities
- Parse and load System 7 sound format from files and memory buffers
- Manage sound sample data storage with format metadata (bit depth, stereo, endianness)
- Organize sound definitions with multiple permutations (variants) per sound
- Provide central sound file interface for opening, loading, and retrieving sound definitions
- Support custom sound slots for runtime-added audio resources
- Cache opened sound files to optimize repeated access

## External Dependencies
- `AStream.h` ΓÇö `AIStreamBE` for big-endian binary deserialization (System 7 format is big-endian)
- `FileHandler.h` ΓÇö `OpenedFile`, `FileSpecifier` for file I/O abstraction
- `<memory>` ΓÇö `std::auto_ptr` for RAII file handle
- `<vector>` ΓÇö `std::vector` for dynamic sound and permutation arrays
- **Defined elsewhere:** `_fixed` type (fixed-point arithmetic for pitch); `OpenedFile`, `FileSpecifier` (file I/O); stream operators in `AIStreamBE`

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

## External Dependencies
- `FIXED_ONE` macro (fixed-point arithmetic constant, defined elsewhere)
- Standard C header guards (`#ifndef`, `#endif`)

---

**Notes:**  
- Comments indicate LP (Linus Pettersson) additions and changes relative to Marathon 2 (M2).
- Commented-out entries (e.g., `_snd_nuclear_hard_death`, `_snd_unused2`) indicate removed or deprecated sounds retained for compatibility.
- The extensive sound taxonomy suggests a game with diverse weapon types, creature AI, environmental simulation, and HUD feedback.

# Source_Files/Sound/VorbisDecoder.cpp
## File Purpose
Implements OGG/Vorbis audio file decoding for the Aleph One game engine. Acts as an SDL-backed decoder that reads Vorbis streams from files and extracts audio frames with metadata (sample rate, stereo/mono).

## Core Responsibilities
- Set up SDL-based I/O callbacks (read, seek, tell, close) for the Vorbis decoder
- Open and validate OGG/Vorbis files, extracting audio metadata (sample rate, channel count)
- Decode compressed audio frames into raw PCM buffers on demand
- Manage file seek/rewind operations
- Clean up Vorbis and file resources on close/destruction

## External Dependencies
- **SDL (Simple DirectMedia Layer):** `SDL_RWops`, `SDL_RWread()`, `SDL_RWseek()`, `SDL_RWtell()`, `SDL_RWclose()` ΓÇô file I/O abstraction
- **libvorbisfile:** `OggVorbis_File`, `ov_test_callbacks()`, `ov_test_open()`, `ov_info()`, `ov_read()`, `ov_raw_seek()`, `ov_clear()`, `ov_callbacks` ΓÇô Vorbis decoding
- **Local headers:** `"VorbisDecoder.h"` (class definition), `"cseries.h"`, `"Decoder.h"` (base class `StreamDecoder`)
- **Conditional compilation:** `#ifdef HAVE_VORBISFILE` (entire file only compiled if Vorbis support is enabled); `#ifdef ALEPHONE_LITTLE_ENDIAN` (endianness query in header)

# Source_Files/Sound/VorbisDecoder.h
## File Purpose
Declares a `VorbisDecoder` class for decoding Ogg/Vorbis compressed audio files in the Aleph One game engine. Inherits from `StreamDecoder` base class to provide a unified interface for audio decoding.

## Core Responsibilities
- Implement Ogg/Vorbis file decoding via libvorbisfile library
- Open, decode, rewind, and close Vorbis audio files
- Report audio format metadata (bit depth, sample rate, stereo, endianness, signed/unsigned)
- Manage decoder state and file I/O callbacks

## External Dependencies
- **cseries.h** ΓÇö core engine types and utilities
- **Decoder.h** ΓÇö abstract `StreamDecoder` base class
- **vorbis/vorbisfile.h** ΓÇö libvorbisfile library (conditional, `HAVE_VORBISFILE` guard)
- **FileHandler.h** ΓÇö `FileSpecifier` type (transitively via Decoder.h)

**Notes:** Compilation requires libvorbisfile dev headers and linking with `-lvorbisfile`. The conditional `#ifdef HAVE_VORBISFILE` allows the engine to build without Vorbis support if the library is unavailable.


