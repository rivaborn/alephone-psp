# Source_Files/Network/network_microphone_shared.cpp

## File Purpose
Utility code for capturing and encoding network microphone audio. Handles format conversion, optional Speex compression, sample-rate interpolation, and packet distribution for real-time voice over the network in Aleph One multiplayer games.

## Core Responsibilities
- Announce and track microphone capture format (sample rate, channels, bit depth)
- Extract audio samples from raw capture buffers with format conversion (mono/stereo, 8/16-bit)
- Perform sample-rate conversion via fixed-point interpolation
- Encode audio frames using Speex codec (if enabled at compile-time)
- Accumulate encoded frames into network packets
- Distribute audio packets to the game network layer

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sSamplesPerSecond` | `uint32` | static | Microphone capture sample rate (Hz) |
| `sStereo` | `uint32` | static | True if capture is stereo; false if mono |
| `s16Bit` | `uint32` | static | True if 16-bit samples; false if 8-bit |
| `sCaptureBytesPerPacket` | `int32` | static | Bytes of raw capture data per network packet |
| `sNumberOfBytesPerSample` | `uint32` | static | Bytes per sample (1ΓÇô4 depending on format) |
| `rate` | `_fixed` | static | Sample-rate conversion ratio (fixed-point) |
| `counter` | `_fixed` | static | Accumulator for sample-rate conversion |

## Key Functions / Methods

### announce_microphone_capture_format
- **Signature:** `bool announce_microphone_capture_format(uint32 inSamplesPerSecond, bool inStereo, bool in16Bit)`
- **Purpose:** Initialize capture format state and compute derived parameters for sample packing and rate conversion.
- **Inputs:** Sample rate (Hz), stereo flag, 16-bit flag.
- **Outputs/Return:** Always `true`.
- **Side effects:** Updates file-static state: `sSamplesPerSecond`, `sStereo`, `s16Bit`, `sNumberOfBytesPerSample`, `sCaptureBytesPerPacket`, `rate`, `counter`.
- **Calls:** None.
- **Notes:** Must be called before any capture processing. Assumes `kNetworkAudioSampleRate` is defined elsewhere (likely 22050 Hz for network audio).

### get_capture_byte_count_per_packet
- **Signature:** `int32 get_capture_byte_count_per_packet()`
- **Purpose:** Return the number of raw capture bytes to collect before forming a network packet.
- **Inputs:** None.
- **Outputs/Return:** Bytes per packet (computed during `announce_microphone_capture_format`).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Asserts `sSamplesPerSecond > 0`; caller must call `announce_microphone_capture_format` first.

### getSample (template)
- **Signature:** `template<bool stereo, bool sixteenBit> inline int16 getSample(void *data)`
- **Purpose:** Extract one audio sample from raw capture data, handling mono/stereo and 8/16-bit conversion, mixing stereo to mono if needed.
- **Inputs:** Pointer to capture data.
- **Outputs/Return:** Signed 16-bit sample.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Stereo samples are averaged; 8-bit samples are unsigned (127 = silence), converted to signed 16-bit.

### copy_and_speex_encode_template
- **Signature:** `template <bool stereo, bool sixteenBit> int32 copy_and_speex_encode_template(uint8* outStorage, void* inStorage, int32 inCount)`
- **Purpose:** Extract audio samples, apply sample-rate conversion via linear interpolation, accumulate into 160-sample Speex frames, and encode frames into network buffer.
- **Inputs:** Output buffer, input raw data, byte count to process.
- **Outputs/Return:** Bytes written to output (includes per-frame size headers).
- **Side effects:** Updates global `counter` (rate-conversion state); resets and writes to `gEncoderBits`/`gEncoderState` (Speex encoder globals).
- **Calls:** `getSample<stereo, sixteenBit>()`, `speex_bits_reset()`, `speex_encode_int()`, `speex_bits_write()`.
- **Notes:** Speex expects 160-sample frames at 8 kHz internal rate. Packs frame size (1 byte) before each encoded frame. Static `storedSamples` and `frame[]` maintain frame state across calls.

### copy_and_speex_encode
- **Signature:** `int32 copy_and_speex_encode(uint8* outStorage, void *inStorage, int32 inCount)`
- **Purpose:** Dispatcher: select template specialization based on capture format flags and delegate to `copy_and_speex_encode_template`.
- **Inputs:** Output buffer, input raw data, byte count.
- **Outputs/Return:** Bytes written.
- **Side effects:** Delegates to template; forwards all side effects.
- **Calls:** `copy_and_speex_encode_template` (4 possible specializations).
- **Notes:** Compile-time specialization eliminates format-checking overhead at runtime.

### send_audio_data
- **Signature:** `static void send_audio_data(void* inData, short inSize)`
- **Purpose:** Distribute an encoded audio packet to other players over the network.
- **Inputs:** Packet data (including network header), packet size.
- **Outputs/Return:** None.
- **Side effects:** Calls either `NetDistributeInformation` (normal) or `received_network_audio_proc` (loopback test mode).
- **Calls:** `NetDistributeInformation` or `received_network_audio_proc`; reads `GET_GAME_OPTIONS()`.
- **Notes:** Conditional `MICROPHONE_LOCAL_LOOPBACK` macro enables testing without network. Passes team-unique-force flag based on game options.

### copy_and_send_audio_data
- **Signature:** `int32 copy_and_send_audio_data(uint8* inFirstChunkReadPosition, int32 inFirstChunkBytesRemaining, uint8* inSecondChunkReadPosition, int32 inSecondChunkBytesRemaining, bool inForceSend)`
- **Purpose:** Main entry point: accumulate audio from two circular-buffer chunks, encode and send complete packets, and return bytes consumed.
- **Inputs:** Two buffer chunks (start, length), force-send flag.
- **Outputs/Return:** Total capture bytes consumed (advance buffer pointers by this amount).
- **Side effects:** Writes to static packet buffer; calls `copy_and_speex_encode` and `send_audio_data`; reads/modifies `network_audio_header`.
- **Calls:** `copy_and_speex_encode`, `send_audio_data`, `netcpy`.
- **Notes:** Handles wrap-around (two chunks); aligns input to sample boundaries; sends when buffer fills or `inForceSend` is true. Only compiled if `SPEEX` defined; else returns sum of both chunks (stub). Asserts `sSamplesPerSecond > 0`.

## Control Flow Notes
Fits into a frame-update or idle-loop routine:
1. Microphone driver captures raw PCM into a circular buffer.
2. Main thread calls `copy_and_send_audio_data(chunk1, len1, chunk2, len2, ...)` periodically to drain buffer.
3. This function encodes and sends packets as they accumulate.
4. Network layer routes packets to other players via `NetDistributeInformation`.
5. Receiver dequeues packets and plays them back via speaker system.

## External Dependencies
- **Includes:** `cseries.h` (base types, macros), `network_speaker_sdl.h`, `network_data_formats.h` (struct serialization), `network_distribution_types.h` (distribution IDs), `network_speex.h` (encoder globals), `map.h` (game options, `GET_GAME_OPTIONS()`).
- **Speex library** (conditional `SPEEX`): `speex/speex.h`.
- **Symbols defined elsewhere:** `kNetworkAudioSampleRate`, `kNetworkAudioBytesPerFrame`, `kNetworkAudioSamplesPerPacket` (constants); `gEncoderState`, `gEncoderBits` (Speex encoder); `NetDistributeInformation`, `netcpy` (network layer); `received_network_audio_proc` (test loopback); `GET_GAME_OPTIONS()`, `_force_unique_teams` (game state).
