# Source_Files/Network/network_microphone_sdl_win32.cpp

## File Purpose
Windows-specific implementation of network microphone capture using DirectX DirectSound API. Manages audio input hardware initialization, circular buffer capture, and transmission of frames over the network for voice communication in multiplayer games.

## Core Responsibilities
- Initialize and manage DirectSound capture buffers and COM interfaces
- Query hardware capabilities and select optimal audio format from preferences
- Implement circular buffer capture and read position tracking
- Transmit captured audio frames to network layer (packeted)
- Manage microphone lifecycle: open, start/stop transmission, close
- Interface with optional Speex audio compression codec
- Provide periodic idle polling for frame transmission

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `CaptureFormat` | struct | Encodes audio format properties: WAVE identifier, sample rate (Hz), stereo flag, 16-bit flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sDirectSoundCapture` | `LPDIRECTSOUNDCAPTURE` | static | COM interface to DirectSound capture device |
| `sCaptureBuffer` | `LPDIRECTSOUNDCAPTUREBUFFER` | static | Circular buffer for captured audio frames |
| `sCaptureBufferSize` | int | static | Size in bytes of the circular capture buffer |
| `sCaptureSystemReady` | bool | static | True if DirectSound init succeeded and format accepted |
| `sNextReadStartPosition` | int | static | Current read offset in circular buffer (modulo buffer size) |
| `sCaptureFormatIndex` | int | static | Index into `sFormatPreferences` of the selected format |
| `sTransmittingAudio` | bool | static | True if capture buffer is actively recording |
| `sFormatPreferences` | `CaptureFormat[]` | static | Array of candidate formats (11 kHz, 22 kHz, 44 kHz; mono/stereo; 8/16-bit); searched in order |

## Key Functions / Methods

### open_network_microphone
- **Signature:** `OSErr open_network_microphone()`
- **Purpose:** Initialize DirectSound capture, select hardware-compatible format, create circular buffer, announce format to platform-agnostic layer.
- **Inputs:** None (reads global format preferences and DirectSound device capabilities)
- **Outputs/Return:** Always returns 0 (OSErr); actual success stored in `sCaptureSystemReady` flag
- **Side effects:** Allocates COM objects (`sDirectSoundCapture`, `sCaptureBuffer`); queries hardware; may initialize Speex encoder if enabled in preferences
- **Calls:** `DirectSoundCaptureCreate()`, `GetCaps()`, `CreateCaptureBuffer()`, `announce_microphone_capture_format()`, `init_speex_encoder()` (conditional), `logAnomaly()`, `logAnomaly1()`, `logAnomaly3()`
- **Notes:** Searches format array in order; only accepts formats with sample rates ΓëÑ network audio rate. Assertions check for double-open.

### transmit_captured_data
- **Signature:** `static void transmit_captured_data(bool inForceSend)`
- **Purpose:** Extract buffered audio from circular capture buffer and pass to network transmission layer.
- **Inputs:** `inForceSend` ΓÇô if true, send partial packet; if false, only send when full packet available
- **Outputs/Return:** None
- **Side effects:** Advances `sNextReadStartPosition`; locks/unlocks capture buffer; calls network layer functions to process audio
- **Calls:** `GetCurrentPosition()`, `Lock()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`, `Unlock()`
- **Notes:** Circular buffer may wrap; code handles two-chunk lock result from DirectSound. Only consumes data if `inForceSend` or enough data for a full packet.

### start_transmitting_audio
- **Signature:** `static void start_transmitting_audio()`
- **Purpose:** Begin recording audio into the circular buffer.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Starts DirectSound capture buffer; sets `sTransmittingAudio` on success
- **Calls:** `sCaptureBuffer->Start(DSCBSTART_LOOPING)`
- **Notes:** Asserts `sCaptureSystemReady` and `!sTransmittingAudio`.

### stop_transmitting_audio
- **Signature:** `static void stop_transmitting_audio()`
- **Purpose:** Stop recording and flush any remaining buffered audio.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Stops capture buffer; clears `sTransmittingAudio` on success; calls `transmit_captured_data(true)` to push final packet
- **Calls:** `sCaptureBuffer->Stop()`, `transmit_captured_data(true)`
- **Notes:** Asserts preconditions.

### close_network_microphone
- **Signature:** `void close_network_microphone()`
- **Purpose:** Release all DirectSound resources and reset module state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Stops transmission if active; releases COM objects; destroys Speex encoder if enabled; clears all static state
- **Calls:** `stop_transmitting_audio()`, `Release()` (on COM objects), `destroy_speex_encoder()` (conditional)
- **Notes:** Safe to call multiple times (uses NULL checks).

### set_network_microphone_state
- **Signature:** `void set_network_microphone_state(bool inActive)`
- **Purpose:** Toggle microphone transmission on/off.
- **Inputs:** `inActive` ΓÇô true to start capturing, false to stop
- **Outputs/Return:** None
- **Side effects:** Calls `start_transmitting_audio()` or `stop_transmitting_audio()` as needed
- **Calls:** `start_transmitting_audio()`, `stop_transmitting_audio()`
- **Notes:** No-op if system not ready or state already matches.

### network_microphone_idle_proc
- **Signature:** `void network_microphone_idle_proc()`
- **Purpose:** Periodic polling routine to transmit buffered audio; expected to be called from main game loop.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** May transmit audio packets
- **Calls:** `transmit_captured_data(false)`
- **Notes:** Only acts if system ready and transmission active.

### is_network_microphone_implemented / has_sound_input_capability
- **Signature:** `bool is_network_microphone_implemented()`, `bool has_sound_input_capability()`
- **Purpose:** Announce that this platform has a real (not dummy) microphone implementation and sound input hardware.
- **Outputs/Return:** Both return `true`
- **Notes:** `has_sound_input_capability()` comment indicates it should actually check hardware; currently hardcoded.

## Control Flow Notes
**Initialization:** `open_network_microphone()` is called once at startup. It initializes DirectSound, discovers the first supported audio format, creates the circular buffer, and announces the format to the platform-agnostic layer (`announce_microphone_capture_format()`).

**Main Loop:** During gameplay, `network_microphone_idle_proc()` is called periodically (likely once per frame). It reads from the circular buffer and submits complete packets to `copy_and_send_audio_data()`.

**State Control:** `set_network_microphone_state()` gates transmission on/off (e.g., push-to-talk or user preference).

**Shutdown:** `close_network_microphone()` stops transmission and releases all COM objects.

**Compression:** If Speex is enabled in preferences, the encoder is initialized after format is set and destroyed at shutdown.

## External Dependencies
- **DirectX SDK:** `<dsound.h>` ΓÇô DirectSoundCapture, DirectSoundCaptureBuffer COM interfaces; DSCBUFFERDESC, WAVEFORMATEX structures
- **Cross-platform layer:** `network_microphone_shared.h` ΓÇô `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- **Platform utilities:** `cseries.h` ΓÇô type definitions, macros
- **Logging:** `Logging.h` ΓÇô `logContext()`, `logAnomaly()`, `logAnomaly1()`, `logAnomaly3()`
- **Preferences:** `preferences.h` ΓÇô `network_preferences` global (Speex encoder flag)
- **Audio compression (optional):** `network_speex.h` ΓÇô `init_speex_encoder()`, `destroy_speex_encoder()` (compiled only if `SPEEX` defined)
- **Speaker layer:** `network_speaker_sdl.h` ΓÇô included but not directly used in this file
