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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Header` | struct | Metadata for audio data: format flags, sample pointers, loop info, sample rate |
| `Channel` | struct | Per-channel playback state: active flag, format, data pointers, volume, pitch, and queued next sound |
| `Mixer` | class | Singleton mixer managing all channels, SDL audio spec, and the mixing algorithm |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `static Mixer*` | static (private) | Singleton instance |
| `local_player_index` | extern short | global | Used to mute audio when local player is transmitting (networked game) |
| `game_is_networked` | extern bool | global | Flag indicating networked multiplayer; affects netmic volume logic |
| `sNetworkAudioBufferDesc` | `NetworkSpeakerSoundBufferDescriptor*` | member | Current network audio buffer being played |

## Key Functions / Methods

### instance()
- Signature: `static Mixer* instance()`
- Purpose: Singleton accessor; creates instance on first call
- Inputs: None
- Outputs/Return: Pointer to static Mixer instance
- Side effects: Allocates Mixer once
- Calls: Constructor (if needed)
- Notes: Thread-unsafe singleton pattern typical of older game code

### Start()
- Signature: `void Start(uint16 rate, bool sixteen_bit, bool stereo, int num_channels, int volume, uint16 samples)`
- Purpose: Initialize SDL audio subsystem and configure output format
- Inputs: Sample rate (Hz), bit depth, channel count, master volume, SDL buffer size
- Outputs/Return: None
- Side effects: Initializes SDL audio, allocates channel array, registers MixerCallback
- Calls: SDL audio init functions (implicit); MixerCallback registration
- Notes: Stores `desired` and `obtained` SDL_AudioSpec for actual format negotiation

### Stop()
- Signature: `void Stop()`
- Purpose: Shut down SDL audio and release resources
- Inputs: None
- Outputs/Return: None
- Side effects: Closes SDL audio device
- Calls: SDL audio close functions (implicit)
- Notes: Called at engine shutdown

### BufferSound()
- Signature: `void BufferSound(int channel, const Header& header, _fixed pitch)`
- Purpose: Queue a sound on a specific channel with optional pitch shift
- Inputs: Channel index, audio format and data metadata, pitch ratio
- Outputs/Return: None
- Side effects: Updates channel state or queues sound for next playback
- Calls: `Channel::BufferSoundHeader()`
- Notes: If channel is currently playing, queues for later; otherwise loads immediately

### SetVolume()
- Signature: `void SetVolume(short volume)`
- Purpose: Set global master volume
- Inputs: Volume level (0x100 = nominal)
- Outputs/Return: None
- Side effects: Updates `main_volume` member
- Calls: None
- Notes: Applied to all channels during mixing

### Music channel methods
- `StartMusicChannel()` ΓÇô Activate music channel with format info
- `UpdateMusicChannel(uint8* data, int len)` ΓÇô Feed decoded music data to channel
- `MusicPlaying()` ΓÇô Return whether music channel is active
- `StopMusicChannel()` ΓÇô Deactivate music (thread-safe with SDL_LockAudio)
- `SetMusicChannelVolume(int16 volume)` ΓÇô Set stereo volume for music
- Notes: Music channel is always at index `sound_channel_count + MUSIC_CHANNEL`

### Network audio methods
- `EnsureNetworkAudioPlaying()` ΓÇô Dequeue and activate network audio if available
- `StopNetworkAudio()` ΓÇô Deactivate network audio channel
- `IsNetworkAudioPlaying()` ΓÇô Check if network channel is active
- Notes: Network channel used for real-time microphone audio in multiplayer

### Mix<T, stereo, is_signed>()
- Signature: `template <class T, bool stereo, bool is_signed> inline void Mix(T *p, int len)`
- Purpose: Core mixing algorithm; fills output buffer by mixing all active channels
- Inputs: Output buffer pointer, sample count
- Outputs/Return: None (writes directly to output buffer `p`)
- Side effects: Advances all active channel playback state (data pointers, counters); dequeues network buffers; manages channel lifecycle (loop/finish/next sound)
- Calls: `Music::FillBuffer()` / `Music::InterruptFillBuffer()`, `SoundManager::GetNetmicVolumeAdjustment()`, `SoundManager::IncrementChannelCallbackCount()`, network speaker functions
- Notes:
  - For each output sample, mixes all active channels using fixed-point pitch adjustment (`counter` accumulates `rate`; when >= 0x10000, advances sample)
  - Performs linear interpolation between current and next sample for smooth pitch shifting
  - Applies per-channel volume and master volume (>> 8 scaling)
  - Reduces non-network audio when network audio is playing (via `GetNetmicVolumeAdjustment()`)
  - Mutes output if local player is transmitting in networked game
  - Clips to signed 16-bit range before conversion to output format
  - Handles channel completion: loops if loop_length > 0; for music/resource/network, dequeues next buffer or deactivates
  - Thread-safety: Called from SDL audio thread; uses SDL_LockAudio for critical sections

### MixerCallback()
- Signature: `static void MixerCallback(void *user, uint8 *stream, int len)`
- Purpose: SDL audio callback; dispatches to instance method
- Inputs: User data (Mixer*), output buffer, byte count
- Outputs/Return: None
- Side effects: Calls instance Callback()
- Calls: `Mixer::Callback()`
- Notes: Static method required by SDL_AudioSpec.callback

### Callback()
- Signature: `void Callback(uint8 *stream, int len)`
- Purpose: Instance audio callback; dispatches to appropriate Mix<> template based on format
- Inputs: Output buffer, byte count
- Outputs/Return: None
- Side effects: Fills output buffer by calling Mix() with format-specific template args
- Calls: Mix<>() with appropriate T, stereo, is_signed template parameters
- Notes: Called on SDL audio thread; must use SDL_LockAudio for shared state

## Control Flow Notes

**Initialization phase**: `Start()` allocates channels, configures SDL audio, and registers the mixer callback with SDL.

**Per-frame audio generation**: SDL audio thread repeatedly calls `MixerCallback()` ΓåÆ `Callback()` ΓåÆ `Mix<>()` to fill output buffer. The mixing loop:
1. Iterates all channels
2. For each active channel, reads and interpolates samples, applies volume, mixes into accumulator
3. Advances playback state (counter, data pointer)
4. On sound completion, loops, queues next sound, or deactivates
5. Applies master volume and muting based on game state
6. Converts to output format and writes to buffer

**Channel lifecycle**: 
- SoundManager queues sounds via `BufferSound()`
- Music class feeds data via `UpdateMusicChannel()` 
- Network speaker pushes audio via `EnsureNetworkAudioPlaying()`
- Mixer automatically deactivates or transitions to next queued sound when current sound finishes

**Shutdown**: `Stop()` closes SDL audio device.

## External Dependencies

- **SDL_endian.h** ΓÇô Endianness conversion macros (SDL_SwapLE16, SDL_SwapBE16)
- **cseries.h** ΓÇô Common type definitions (_fixed, uint8, int16, etc.)
- **network_speaker_sdl.h** ΓÇô NetworkSpeakerSoundBufferDescriptor, dequeue_network_speaker_data(), release_network_speaker_buffer()
- **network_audio_shared.h** ΓÇô Network audio format constants
- **map.h** ΓÇô External dynamic_world (accessed for speaking_player_index during muting logic)
- **Music.h** ΓÇô Music::instance(), FillBuffer(), InterruptFillBuffer() 
- **SoundManager.h** ΓÇô SoundManager::instance(), GetNetmicVolumeAdjustment(), IncrementChannelCallbackCount()
