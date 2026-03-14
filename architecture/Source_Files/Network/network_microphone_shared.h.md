# Source_Files/Network/network_microphone_shared.h

## File Purpose
Defines the shared interface for network microphone implementations in Marathon: Aleph One. Provides three key functions for announcing audio capture format, buffering and sending audio data, and querying packet size requirements. Internal use only by netmic implementation files.

## Core Responsibilities
- Announce microphone capture format (sample rate, stereo/mono, bit depth) and track it internally
- Buffer captured audio data and determine when sufficient data exists to send a network packet
- Send audio data over the network, respecting format and buffering constraints
- Provide queries for packet size calculations to guide preprocessing decisions

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| (implicit capture format state) | format parameters (sample rate, stereo, bit depth) | static (internal) | Caches the most recently announced format for use by `copy_and_send_audio_data()` and `get_capture_byte_count_per_packet()` |

## Key Functions / Methods

### announce_microphone_capture_format
- **Signature:** `bool announce_microphone_capture_format(uint32 inSamplesPerSecond, bool inStereo, bool in16Bit)`
- **Purpose:** Register the audio capture format and validate it for network transmission.
- **Inputs:** Sample rate (Hz), stereo flag, 16-bit flag
- **Outputs/Return:** Boolean indicating whether the format is usable
- **Side effects:** Updates internal global format state; affects behavior of `copy_and_send_audio_data()` and `get_capture_byte_count_per_packet()`
- **Calls:** Not inferable from this file
- **Notes:** Must be called before `copy_and_send_audio_data()`. Should be called again if capture format changes at runtime.

### copy_and_send_audio_data
- **Signature:** `int32 copy_and_send_audio_data(uint8* inFirstChunkReadPosition, int32 inFirstChunkBytesRemaining, uint8* inSecondChunkReadPosition, int32 inSecondChunkBytesRemaining, bool inForceSend)`
- **Purpose:** Buffer incoming audio data and send network packets when sufficient data accumulates or force-send is requested.
- **Inputs:** Two chunks of captured audio (pointer + remaining bytes), force-send flag
- **Outputs/Return:** Number of bytes consumed
- **Side effects:** Sends network packets; updates internal buffer state
- **Calls:** Not inferable from this file
- **Notes:** Does not consume data unless buffered amount reaches packet threshold or `inForceSend` is true. Requires prior call to `announce_microphone_capture_format()`.

### get_capture_byte_count_per_packet
- **Signature:** `int32 get_capture_byte_count_per_packet()`
- **Purpose:** Query the buffer size needed to fill one network packet at the current capture format.
- **Inputs:** None
- **Outputs/Return:** Byte count required per packet
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** Return value is valid only for the most recently announced format. Calling without prior `announce_microphone_capture_format()` is an error.

## Control Flow Notes
Expected usage sequence:
1. Call `announce_microphone_capture_format()` once during initialization or when format changes
2. In capture loop: optionally call `get_capture_byte_count_per_packet()` to decide if preprocessing is worthwhile
3. Repeatedly call `copy_and_send_audio_data()` with captured chunks
4. On shutdown/flush: call `copy_and_send_audio_data()` with `inForceSend=true`

## External Dependencies
- Platform integer types: `uint32`, `uint8`, `int32` (defined elsewhere)
- Callers are netmic implementation files; function implementations are defined elsewhere in the network module
