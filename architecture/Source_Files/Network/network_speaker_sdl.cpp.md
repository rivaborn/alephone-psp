# Source_Files/Network/network_speaker_sdl.cpp

## File Purpose
Implements network audio playback for SDL platforms in the Aleph One game engine. Manages incoming network audio buffers, generates noise for buffering smoothness, and coordinates with the audio mixer for real-time playback during networked games.

## Core Responsibilities
- Allocate and manage audio data buffer pools for incoming network audio
- Generate and queue noise buffers to smooth audio playback during buffering
- Coordinate buffer lifecycle between network receiver and audio mixer threads
- Track speaker state (active/inactive) and handle dry-queue edge cases
- Initialize Speex decoder support when enabled
- Clean up resources on shutdown

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `NetworkSpeakerSoundBufferDescriptor` | struct (defined in header) | Describes an audio buffer: pointer, length, and disposability flags |
| `CircularQueue<NetworkSpeakerSoundBufferDescriptor>` | template instance | Queue of sound descriptors sent to the mixer |
| `CircularQueue<byte*>` | template instance | Free pool of reusable data buffers for incoming audio |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSoundBuffers` | `CircularQueue<NetworkSpeakerSoundBufferDescriptor>` | static | Queued audio buffers awaiting playback |
| `sSoundDataBuffers` | `CircularQueue<byte*>` | static | Pool of available buffers for network audio data |
| `sNoiseBufferStorage` | `byte*` | static | Pre-generated white noise used during buffering |
| `sNoiseBufferDesc` | `NetworkSpeakerSoundBufferDescriptor` | static | Descriptor for noise buffer |
| `sDryDequeues` | `int` | static | Counter of consecutive empty-queue dequeues |
| `sSpeakerIsOn` | `bool` | static | Flag: speaker is currently active/playing |

## Key Functions / Methods

### open_network_speaker
- **Signature:** `OSErr open_network_speaker()`
- **Purpose:** Initialize network speaker system; allocate noise and data buffers.
- **Inputs:** None
- **Outputs/Return:** `OSErr` (0 on success)
- **Side effects:** Allocates heap memory for noise buffer (`kNoiseBufferSize` bytes) and data buffers (`kNumSoundDataBuffers ├ù kSoundDataBufferSize`); initializes all queues and state variables; optionally calls `init_speex_decoder()`.
- **Calls:** `local_random()`, `sSoundBuffers.reset()`, `sSoundDataBuffers.reset()`, `init_speex_decoder()` (conditional)
- **Notes:** Noise buffer is allocated once and reused. Data buffers are pre-allocated to avoid allocations during playback. Idempotent for noise buffer (checks `sNoiseBufferStorage != NULL`).

### queue_network_speaker_data
- **Signature:** `void queue_network_speaker_data(byte* inData, short inLength)`
- **Purpose:** Queue incoming network audio data for playback.
- **Inputs:** `inData` (audio bytes), `inLength` (byte count)
- **Outputs/Return:** None
- **Side effects:** Dequeues a free buffer from `sSoundDataBuffers`, copies audio data, marks buffer as disposable, enqueues descriptor to `sSoundBuffers`. On first data, primes queue with `kNumPumpPrimes` noise buffers and sets `sSpeakerIsOn = true`.
- **Calls:** `memcpy()`, `sSoundDataBuffers.peek()/.dequeue()`, `sSoundBuffers.enqueue()`, `fdprintf()` (error only)
- **Notes:** Silently discards data if no free buffers available; prints warning but does not assert. Noise priming ensures smooth audio start during buffering.

### network_speaker_idle_proc
- **Signature:** `void network_speaker_idle_proc()`
- **Purpose:** Periodic maintenance callback from main thread.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Mixer::instance()->EnsureNetworkAudioPlaying()` if speaker is active.
- **Calls:** `Mixer::instance()->EnsureNetworkAudioPlaying()`
- **Notes:** Lightweight; ensures mixer is aware of pending network audio.

### dequeue_network_speaker_data
- **Signature:** `NetworkSpeakerSoundBufferDescriptor* dequeue_network_speaker_data()`
- **Purpose:** Retrieve next audio buffer for playback (called by mixer).
- **Inputs:** None
- **Outputs/Return:** Pointer to static buffer descriptor; `NULL` if audio dried up.
- **Side effects:** Dequeues from `sSoundBuffers` if available; increments `sDryDequeues` on empty queue. Sets `sSpeakerIsOn = false` after `kMaxDryDequeues` consecutive empty dequeues. Returns pointer to static `sBufferDesc` (invalidated on next call).
- **Calls:** `sSoundBuffers.getCountOfElements()/.peek()/.dequeue()`
- **Notes:** Caller must not retain returned pointer across calls. Fills with noise buffers during droughts to smooth audio. Edge case: after exactly `kMaxDryDequeues` empty dequeues, returns `NULL` and disables speaker.

### close_network_speaker
- **Signature:** `void close_network_speaker()`
- **Purpose:** Shut down network speaker system and release all resources.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Mixer::instance()->StopNetworkAudio()`; drains `sSoundBuffers` and releases disposable buffers; deletes all data buffers from `sSoundDataBuffers`; deletes noise buffer; resets state variables; optionally calls `destroy_speex_decoder()`.
- **Calls:** `Mixer::instance()->StopNetworkAudio()`, `dequeue_network_speaker_data()`, `is_sound_data_disposable()`, `release_network_speaker_buffer()`, `destroy_speex_decoder()` (conditional)
- **Notes:** Safe to call multiple times; critical to stop mixer before draining queues to avoid race conditions.

### release_network_speaker_buffer
- **Signature:** `void release_network_speaker_buffer(byte* inBuffer)`
- **Purpose:** Return a used audio buffer to the free pool.
- **Inputs:** `inBuffer` (pointer previously obtained via `dequeue_network_speaker_data`)
- **Outputs/Return:** None
- **Side effects:** Enqueues buffer to `sSoundDataBuffers`.
- **Calls:** `sSoundDataBuffers.enqueue()`
- **Notes:** Called by mixer after consuming a buffer marked disposable.

## Control Flow Notes
1. **Initialization:** `open_network_speaker()` allocates buffers and state on main thread.
2. **Reception:** Main thread (or network thread) calls `queue_network_speaker_data()` with incoming audio chunks; first chunk primes queue with noise.
3. **Playback:** Mixer (audio thread) calls `network_speaker_idle_proc()` (main thread) and `dequeue_network_speaker_data()` (audio callback) on demand.
4. **Buffer Recycling:** Mixer calls `release_network_speaker_buffer()` after consuming disposable buffers, returning them to the free pool.
5. **Shutdown:** `close_network_speaker()` drains queues and releases heap memory on main thread.

## External Dependencies
- **network_sound.h, network_speaker_sdl.h** ΓÇö Interface definitions for speaker API.
- **network_distribution_types.h** ΓÇö Network distribution type constants (unused in this file).
- **CircularQueue.h** ΓÇö Template-based queue container.
- **world.h** ΓÇö `local_random()` for noise generation.
- **Mixer.h** ΓÇö `Mixer::instance()` singleton for audio system integration.
- **network_speex.h** (conditional) ΓÇö Speex codec decoder initialization/destruction when `SPEEX` defined.
