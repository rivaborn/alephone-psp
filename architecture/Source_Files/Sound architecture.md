# Subsystem Overview

## Purpose

The Sound subsystem provides audio playback and mixing for the Aleph One game engine, handling multi-channel real-time audio mixing, spatial audio calculations, and support for multiple compressed and uncompressed audio formats. It coordinates between SDL's cross-platform audio I/O, game world state (listener position, obstruction), and both built-in (System 7 format) and external audio resources.

## Key Files

| File | Role |
|------|------|
| SoundManager.cpp/h | Core sound subsystem singleton; coordinates loading, playback, channel allocation, spatial audio calculations, and parameter configuration |
| Mixer.cpp/h | Real-time audio mixing engine; SDL audio callback handler; manages multiple concurrent channels with format conversion and volume interpolation |
| Music.cpp/h | Singleton music playback manager; handles intro and level music with fade effects, playlist support, and buffer refilling |
| SoundFile.cpp/h | System 7 sound format parser and loader; manages built-in sound definitions, permutations, and custom sound slots |
| Decoder.cpp/h | Factory pattern for audio format decoders; instantiates format-specific decoders (MAD, Vorbis, libsndfile, BasicIFF) |
| BasicIFFDecoder.cpp/h | Decoder for uncompressed AIFF/WAV files; extracts format metadata (sample rate, channels, bit depth, endianness) |
| MADDecoder.cpp/h | MP3 decoder via libmad; converts fixed-point audio to PCM 16-bit signed integers with endianness handling |
| VorbisDecoder.cpp/h | OGG/Vorbis decoder via libvorbisfile; SDL-backed I/O callbacks for stream access |
| SndfileDecoder.cpp/h | Multipurpose decoder via libsndfile (supports WAV, FLAC, OGG Vorbis); streaming PCM interface |
| ReplacementSounds.cpp/h | Hash-table-based external sound replacement system; O(1) lookup for runtime audio overrides by (Index, Slot) |
| SoundManagerEnums.h | Sound ID enumerations (ambient, random, triggered sounds), behavior profiles, initialization flags, obstruction constants |
| sound_definitions.h | Static sound metadata (behavior profiles with distance-based attenuation curves, per-sound properties) |
| song_definitions.h | Song metadata structures with segment boundaries (intro, chorus, trailer) and looping behavior |

## Core Responsibilities

- Load and decode audio in multiple formats (WAV, AIFF, MP3 via libmad, Ogg/Vorbis, FLAC via libsndfile) via format-agnostic Decoder factory
- Initialize SDL audio subsystem with configurable sample rate, bit depth, and channel count; manage audio device lifecycle
- Real-time mixing of multiple concurrent audio channels (effects, music, network audio) with linear interpolation for pitch shifting
- Execute format conversion during mixing (8/16-bit, mono/stereo, signed/unsigned, little/big-endian byte-order adaptation)
- Calculate 3D spatial audio: stereo panning, distance-based volume attenuation, listener obstruction detection
- Allocate mixer channels intelligently using priority queues and conflict detection; map sounds to available channels
- Manage music playback independent of effects with fade-out effects and level-specific playlists
- Support external sound file replacement via hash-table lookup for runtime audio customization
- Coordinate network microphone audio dequeue and playback with dynamic buffer lifecycle and local-transmission muting
- Parse System 7 big-endian sound headers (22-byte standard and 64-byte extended formats)
- Maintain ambient and random sound sources with active culling and enumeration callbacks

## Key Interfaces & Data Flow

**Subsystem exposes:**
- SoundManager singleton: `Initialize()`, `Shutdown()`, `LoadSoundFile()`, `PlaySound()`, `StopSound()`, volume queries
- Mixer singleton: SDL audio callback interface, channel state management
- Music singleton: intro/level music playback lifecycle with fade effects
- Decoder factory: format auto-detection and instantiation via `Decoder::Get()`
- ReplacementSounds singleton: O(1) lookup by (Index, Slot) for external sound overrides

**Subsystem consumes:**
- SDL audio subsystem: `SDL_OpenAudio()`, `SDL_CloseAudio()`, `SDL_PauseAudio()`, `SDL_LockAudio()`, audio callback context
- Game world: listener position/orientation via `_sound_listener_proc()`, obstruction queries via `_sound_obstructed_proc()`, ambient source enumeration via `_sound_add_ambient_sources_proc()`
- Sound resource files: System 7 `.snd2` files with built-in sound definitions and permutations
- External sound files: WAV, AIFF, MP3, Vorbis, FLAC from disk via FileSpecifier
- XML configuration: sound remapping and customization via `XML_ElementParser` subclass
- Network audio: dequeue API (`dequeue_network_speaker_data()`, `release_network_speaker_buffer()`)

## Runtime Role

- **Initialization**: `SoundManager::Initialize()` opens SDL audio device, initializes `Mixer` with SDL spec callback, loads built-in `.snd2` file
- **Frame (per-SDL-audio-callback)**: `Mixer` callback executes; queries `Music::instance()->FillBuffer()` for music data, dequeues network microphone audio, performs real-time sample mixing from all active channels with format conversion, applies master volume and per-channel muting
- **Music buffer refills**: `Music::UpdateMusicChannel()` called during mixer callback to refill music buffer from active `StreamDecoder`
- **Shutdown**: `SoundManager::Shutdown()` closes SDL audio device, unloads sound definitions, clears replacement sounds, stops all active playback

## Notable Implementation Details

- **Streaming decoders**: `StreamDecoder` abstract interface supports memory-efficient playback of large files without loading entire audio into memory
- **Format-agnostic factory**: `Decoder::Get()` attempts decoders in priority order (libsndfile ΓåÆ Vorbis ΓåÆ libmad ΓåÆ BasicIFF); first successful `Open()` wins; enables conditional compilation for codec availability via `HAVE_SNDFILE`, `HAVE_VORBISFILE`, `HAVE_MAD` build flags
- **Music independent path**: Music buffer lifecycle separate from effects channels; managed by `Music` singleton with fade interpolation (`_fixed` math) and platform-specific handling (macOS interrupt-safe updates noted)
- **Network audio integration**: `kNetworkAudioSampleRate`, `kNetworkAudioIs16Bit`, `kNetworkAudioIsStereo` constants define network format; dynamic buffer dequeue during mixer callback; local player microphone muted during transmission
- **Spatial audio**: 3D vector math (`distance3d()`, `arctangent()`) calculates panning and volume for sounds positioned in game world; distance-based attenuation curves defined per-sound behavior profile
- **System 7 compatibility**: Big-endian header parsing with `AIStreamBE` stream reader; supports loop points, multiple permutations per sound, sample-rate variants
- **External sound replacement**: `ExternalSoundHeader` loads decoder-backed audio; hash table with linear-search fallback for O(1) retrieval; enables player-provided audio without modifying `.snd2` resources
- **Multi-codec output format**: Mixer handles 8/16-bit signed/unsigned mono/stereo from any decoder into single output format via endianness conversion macros (`SDL_SwapLE16`, `SDL_SwapBE16`)
