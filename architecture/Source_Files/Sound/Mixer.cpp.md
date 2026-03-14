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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `Header` | struct | Audio metadata: format flags, sample data pointer, loop boundaries, playback rate |
| `Channel` | struct | Per-channel playback state: format, data pointers, playback counters, volume, queued sound |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `Mixer::m_instance` | `Mixer*` | static | Singleton instance for global access |
| `option_nosound` | `bool` | extern | External flag to disable audio system entirely |
| `sNetworkAudioBufferDesc` | `NetworkSpeakerSoundBufferDescriptor*` | member | Current network audio buffer being decoded |

## Key Functions / Methods

### Start
- **Signature:** `void Start(uint16 rate, bool sixteen_bit, bool stereo, int num_channels, int volume, uint16 samples)`
- **Purpose:** Initialize SDL audio subsystem with desired audio format and open audio device
- **Inputs:** Sample rate, bit depth, channel layout, sound channel count, master volume, buffer sample size
- **Outputs/Return:** None; sets up `channels` vector and `desired`/`obtained` SDL specs
- **Side effects:** Opens SDL audio device; calls `alert_user()` if audio init fails; may skip audio if `option_nosound` set
- **Calls:** `SDL_OpenAudio()`, `SDL_PauseAudio()`, `alert_user()`
- **Notes:** If audio device fails, `sound_channel_count` is set to 0 to disable all mixing; `obtained` spec reflects actual device capabilities

### Stop
- **Signature:** `void Stop()`
- **Purpose:** Shut down audio subsystem and clear channel state
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes SDL audio device; deallocates `channels` vector
- **Calls:** `SDL_CloseAudio()`

### BufferSound
- **Signature:** `void BufferSound(int channel, const Header& header, _fixed pitch)`
- **Purpose:** Queue a sound to a channel (either load directly or append to queue)
- **Inputs:** Channel index, sound header metadata, playback pitch multiplier
- **Outputs/Return:** None
- **Side effects:** Locks audio to prevent race during state update; either calls `LoadSoundHeader()` directly or queues via `BufferSoundHeader()`
- **Calls:** `SDL_LockAudio()`, `Channel::LoadSoundHeader()`, `Channel::BufferSoundHeader()`, `SDL_UnlockAudio()`
- **Notes:** Uses mutex lock because channel state is accessed both by caller and SDL callback thread

### Callback / MixerCallback
- **Signature:** `void Callback(uint8 *stream, int len)` (static wrapper `MixerCallback`)
- **Purpose:** SDL audio callback invoked regularly to fill output buffer; dispatches to template mixer based on audio format
- **Inputs:** Output buffer pointer, buffer length in bytes
- **Outputs/Return:** Fills `stream` with mixed audio samples
- **Side effects:** No locking (callback is single-threaded); reads from all active channels
- **Calls:** `Mix<T, stereo, is_signed>()` template (format-specific mixing)
- **Notes:** Detects format from `obtained` spec and selects appropriate template instantiation; divides buffer length by sample size

### Mix (template in header)
- **Signature:** `template <class T, bool stereo, bool is_signed> inline void Mix(T *p, int len)`
- **Purpose:** Core mixing loop: iterate output samples, read/interpolate from all active channels, apply volume and clipping
- **Inputs:** Output buffer typed to sample format, sample count
- **Outputs/Return:** Writes mixed samples to output buffer in-place
- **Side effects:** Advances `counter`, `data`, `length` in active channels; may deactivate channels on completion; queues callbacks for `SoundManager`
- **Calls:** `Channel::LoadSoundHeader()`, `Music::instance()->FillBuffer()`, `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`
- **Notes:** Reads current and next sample from each channel, interpolates based on fractional counter; handles looping and queued sound transitions; applies netmic volume ducking when network audio active; clips to 16-bit range then converts to output format

### StartMusicChannel / UpdateMusicChannel
- **Signature:** `void StartMusicChannel(...)` / `void UpdateMusicChannel(uint8* data, int len)`
- **Purpose:** Initialize music channel with format and activate; update with new data buffer
- **Inputs:** Audio format flags, sample rate, data pointer, length
- **Outputs/Return:** None
- **Side effects:** Directly modifies `channels[sound_channel_count + MUSIC_CHANNEL]` state
- **Calls:** None (direct member access)
- **Notes:** Music channel is always active after `StartMusicChannel()`; uses special handling in mixer callback to refill via `Music::FillBuffer()`

### EnsureNetworkAudioPlaying / StopNetworkAudio
- **Signature:** `void EnsureNetworkAudioPlaying()` / `void StopNetworkAudio()`
- **Purpose:** Activate network audio channel with dequeued network buffer; deactivate and release buffer
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Locks audio; dequeues or releases `sNetworkAudioBufferDesc`; initializes or clears `channels[sound_channel_count + NETWORK_AUDIO_CHANNEL]`
- **Calls:** `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`
- **Notes:** Network audio channel is special-cased in mixer to apply netmic volume and auto-dequeue next buffer on end

### PlaySoundResource / StopSoundResource
- **Signature:** `void PlaySoundResource(LoadedResource &rsrc)` / `void StopSoundResource()`
- **Purpose:** Parse and play a Mac-format sound resource from memory; stop resource playback
- **Inputs:** Resource object containing sound resource data
- **Outputs/Return:** None
- **Side effects:** Locks audio; reads/parses resource format; loads sound header into resource channel
- **Calls:** `SDL_RWFromMem()`, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SDL_RWseek()`, `SDL_RWclose()`, `SoundHeader::Load()`, `Channel::LoadSoundHeader()`
- **Notes:** Parses Mac sound resource format (format 1/2), scans for `0x8051` buffer command, extracts embedded `SoundHeader`

### Channel::LoadSoundHeader
- **Signature:** `void LoadSoundHeader(const Header& header, _fixed pitch)`
- **Purpose:** Copy sound metadata into channel and compute effective playback rate
- **Inputs:** Header struct with format and data, pitch multiplier
- **Outputs/Return:** None
- **Side effects:** Overwrites channel state; resets `counter` to 0
- **Calls:** `instance()->obtained.freq` (read device sample rate)
- **Notes:** Computes `rate = (pitch >> 8) * ((header.rate >> 8) / device_rate)` for sample-rate conversion

## Control Flow Notes
The mixer operates in a callback-driven model: SDL calls `Callback()` at regular intervals (typically 50ΓÇô200 Hz depending on buffer size). Each callback invocation:
1. Iterates through all samples in the output buffer
2. For each sample, loops through all channels and reads/interpolates audio
3. Accumulates left/right samples with per-channel volume
4. On channel completion, handles looping, music refill, network audio dequeue, or queued sound transitions
5. Applies master volume, network muting, and output clipping
6. Converts to output format and writes to buffer

This design allows decoupling sound scheduling (via `BufferSound`, music updates) from actual audio production, which happens asynchronously in SDL's audio thread.

## External Dependencies
- **SDL:** `SDL_OpenAudio()`, `SDL_CloseAudio()`, `SDL_PauseAudio()`, `SDL_LockAudio()`, `SDL_UnlockAudio()`, `SDL_RWFromMem()`, `SDL_RWops`, `SDL_ReadBE16()`, `SDL_ReadBE32()`, `SDL_RWseek()`, `SDL_RWclose()`, `SDL_SwapLE16()`, `SDL_SwapBE16()`, `SDL_AudioSpec`
- **Music subsystem:** `Music::instance()->FillBuffer()`, `Music::instance()->InterruptFillBuffer()`
- **SoundManager:** `SoundManager::instance()->GetNetmicVolumeAdjustment()`, `SoundManager::instance()->IncrementChannelCallbackCount()`
- **Network audio:** `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`, `kNetworkAudioIsStereo`, `kNetworkAudioIs16Bit`, `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame`, `kNetworkAudioIsSigned8Bit`
- **interface.h:** `strERRORS`, `badSoundChannels`, `alert_user()`, `infoError`
- **Global state:** `game_is_networked`, `local_player_index`, `dynamic_world->speaking_player_index`
- **SoundHeader class:** Loaded from resources; provides format and sample data
