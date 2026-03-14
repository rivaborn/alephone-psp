# Source_Files/Network/network_microphone_coreaudio.cpp

## File Purpose
Implements audio capture from the system microphone on macOS using CoreAudio. Captures raw audio samples from the default input device, buffers them, and forwards to a network transmission pipeline. Conditionally includes Speex compression support.

## Core Responsibilities
- Initialize CoreAudio HAL (Hardware Abstraction Layer) for audio input
- Enumerate sample rates and configure the input device to a compatible rate
- Set up audio stream format conversion (to mono, 16-bit PCM)
- Allocate and manage audio buffers for captured frames
- Implement input callback to receive audio data from the hardware
- Buffer incoming samples and batch-send when a packet threshold is reached
- Control microphone state (start/stop capture)
- Release all CoreAudio resources on shutdown

## Key Types / Data Structures
None defined in this file. Uses external CoreAudio types: `AudioUnit`, `AudioDeviceID`, `AudioStreamBasicDescription`, `AudioBufferList`, `AURenderCallbackStruct`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fAudioUnit` | `AudioUnit` | static | Handle to the CoreAudio input unit |
| `fInputDeviceID` | `AudioDeviceID` | static | System default input device identifier |
| `fOutputFormat` | `AudioStreamBasicDescription` | static | Target format (mono, 16-bit, 1 frame/packet) |
| `fDeviceFormat` | `AudioStreamBasicDescription` | static | Device's native audio format |
| `fAudioSamples` | `Uint32` | static | Number of frames per IO buffer callback |
| `fAudioBuffer` | `AudioBufferList*` | static | Allocated buffer for hardware renders |
| `initialized` | `bool` | static | Tracks whether open_network_microphone() succeeded |
| `captureBuffer` | `std::vector<uint8>` | static | Ring buffer accumulating samples until send threshold |
| `captureBufferSize` | `Uint32` | static | Current occupancy of captureBuffer |
| `mic_active` | `bool` | static | Whether AudioUnit is running |

## Key Functions / Methods

### audio_input_proc
- **Signature:** `OSStatus audio_input_proc(void *, AudioUnitRenderActionFlags *ioActionFlags, const AudioTimeStamp *inTimeStamp, UInt32 inBusNumber, UInt32 inNumberFrames, AudioBufferList *)`
- **Purpose:** CoreAudio callback invoked when input samples are ready. Renders hardware data into fAudioBuffer, buffers it, and triggers network send when packet size threshold is met.
- **Inputs:** inTimeStamp (unused), inBusNumber (always 1ΓÇöinput bus), inNumberFrames (samples ready), ioActionFlags, AudioBufferList (unused parameter).
- **Outputs/Return:** OSStatus (noErr or render error code).
- **Side effects:** Renders into `fAudioBuffer`, accumulates bytes into `captureBuffer`, calls `copy_and_send_audio_data()` when threshold reached, modifies `captureBufferSize`.
- **Calls:** `AudioUnitRender()`, `memcpy()`, `get_capture_byte_count_per_packet()`, `copy_and_send_audio_data()`.
- **Notes:** Assumes `fAudioBuffer` is pre-allocated; inNumberFrames is always Γëñ fAudioSamples; copies inNumberFrames*2 bytes (16-bit mono).

### open_network_microphone
- **Signature:** `OSErr open_network_microphone()`
- **Purpose:** Full initialization: find/open CoreAudio HAL output unit, enable input, select default device, iterate sample rates, configure format, allocate buffers, register callback, initialize Speex if enabled.
- **Inputs:** None.
- **Outputs/Return:** `noErr` on success, error code (e.g., from AudioUnit calls) on failure.
- **Side effects:** Populates all file-static AudioUnit/device state; allocates fAudioBuffer and captureBuffer; calls `AudioUnitInitialize()`; calls `announce_microphone_capture_format()` and conditionally `init_speex_encoder()`; sets `initialized = true`.
- **Calls:** `FindNextComponent()`, `OpenAComponent()`, `AudioUnitSetProperty()` (6 calls), `AudioHardwareGetProperty()`, `AudioDeviceSetProperty()` (in loop), `AudioUnitGetProperty()` (2 calls), `AudioUnitInitialize()`, `calloc()`, `malloc()`, `announce_microphone_capture_format()`, `get_capture_byte_count_per_packet()`, `init_speex_encoder()` (ifdef SPEEX).
- **Notes:** Tries sample rates [8000, 48000, 44100, 22050, 11025]; silently continues if device fails a rate. Assumes kAudioUnitProperty_EnableIO on scope_Input bus 1 is the input channel. Converts device format to fixed mono 16-bit output. If announce_microphone_capture_format() returns false, returns -1 instead of OSErr. captureBuffer is oversized by one frame to handle wraparound.

### set_network_microphone_state
- **Signature:** `void set_network_microphone_state(bool inActive)`
- **Purpose:** Start or stop audio capture by calling AudioOutputUnitStart/Stop if not already in the requested state.
- **Inputs:** `inActive` (true to start, false to stop).
- **Outputs/Return:** None.
- **Side effects:** Calls `AudioOutputUnitStart()` or `AudioOutputUnitStop()`; toggles `mic_active`.
- **Calls:** `AudioOutputUnitStart()` or `AudioOutputUnitStop()`.
- **Notes:** Silently does nothing if `!initialized`.

### close_network_microphone
- **Signature:** `void close_network_microphone()`
- **Purpose:** Release all allocated resources and destroy Speex encoder.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Frees fAudioBuffer and its mData pointer, calls `destroy_speex_encoder()` (ifdef SPEEX), sets `initialized = false`.
- **Calls:** `free()`, `destroy_speex_encoder()` (ifdef SPEEX).
- **Notes:** Safe to call even if initialization failed (fAudioBuffer may be NULL).

### is_network_microphone_implemented
- **Signature:** `bool is_network_microphone_implemented()`
- **Purpose:** Stub returning true; indicates CoreAudio backend is available.
- **Inputs:** None.
- **Outputs/Return:** `true`.

### network_microphone_idle_proc
- **Signature:** `void network_microphone_idle_proc()`
- **Purpose:** Called periodically by the game loop; currently a no-op (CoreAudio is fully interrupt-driven).
- **Inputs:** None.
- **Outputs/Return:** None.

## Control Flow Notes
**Init:** `open_network_microphone()` discovers and configures the CoreAudio HAL once. **Frame:** `set_network_microphone_state()` toggles audio unit on/off per game state. **Update:** `network_microphone_idle_proc()` is called each frame but does nothing (all I/O is asynchronous in the callback). **Capture:** `audio_input_proc()` fires asynchronously whenever the hardware has new samples; it buffers and batches sends. **Shutdown:** `close_network_microphone()` releases memory and destroys compression state.

## External Dependencies
- **Includes:** `<Carbon/Carbon.h>` (macOS CoreAudio types), `<AudioUnit/AudioUnit.h>` (AudioUnit API).
- **Defined elsewhere:** `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()` (from network_microphone_shared.h), `init_speex_encoder()`, `destroy_speex_encoder()` (ifdef SPEEX, implementation location unknown).
