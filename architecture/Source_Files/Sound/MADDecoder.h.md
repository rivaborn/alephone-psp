# Source_Files/Sound/MADDecoder.h

## File Purpose
Declares the `MADDecoder` class, an MP3 audio decoder that wraps the libmad library. Enables the engine to decode and stream MP3 files, providing access to audio samples with format metadata (channels, bit depth, sample rate).

## Core Responsibilities
- Opens and manages MP3 file streams via libmad
- Decodes MP3 frames and produces raw PCM audio samples
- Provides format queries (stereo, 16-bit, signed, sample rate, endianness)
- Maintains audio decoding state (current frame, synthesis buffers, input stream)
- Handles file I/O buffering and rewind/close operations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `MADDecoder` | class | MP3 decoder inheriting `StreamDecoder` interface; wraps libmad decoding pipeline |
| `mad_stream` | struct (libmad) | Input stream state for libmad parser |
| `mad_frame` | struct (libmad) | Decoded MP3 frame header and data |
| `mad_synth` | struct (libmad) | PCM synthesis state (decompressed audio) |
| `OpenedFile` | typedef (defined elsewhere) | File handle wrapper |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `INPUT_BUFFER_SIZE` | int (const) | static | Input buffer capacity (40 KB + guard) for MP3 stream data |
| `stereo` | bool | instance | Tracks if audio is stereo (vs. mono) |
| `bytes_per_frame` | int | instance | Bytes per PCM sample frame (e.g., 4 for stereo 16-bit) |
| `rate` | float | instance | Sample rate in Hz (e.g., 44100) |
| `sample` | int | instance | Current sample index within decoded frame |
| `file` | OpenedFile | instance | Open file handle |
| `file_done` | bool | instance | Flag: end of file reached |
| `InputBuffer` | uint8[40968] | instance | MP3 compressed data buffer + guard space |

## Key Functions / Methods

### Open
- **Signature:** `bool Open(FileSpecifier &File)`
- **Purpose:** Opens an MP3 file and initializes libmad decoder state
- **Inputs:** File specifier (path/handle)
- **Outputs/Return:** `true` on success, `false` on failure
- **Side effects:** Allocates/initializes `Stream`, `Frame`, `Synth`; opens file; sets `stereo`, `bytes_per_frame`, `rate`
- **Calls:** (not visible in header)
- **Notes:** Reads first frame to determine audio format

### Decode
- **Signature:** `int32 Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decodes MP3 data and writes PCM samples to output buffer
- **Inputs:** Output buffer pointer, max bytes to write
- **Outputs/Return:** Bytes written (0 if end of stream)
- **Side effects:** Advances `sample` and frame state; reads from file into `InputBuffer`
- **Calls:** `DecodeFrame()` (private)
- **Notes:** Handles partial frames (multiple calls may be needed for one MP3 frame)

### Rewind
- **Signature:** `void Rewind()`
- **Purpose:** Resets playback to file start
- **Side effects:** Resets libmad state and file position

### Close
- **Signature:** `void Close()`
- **Purpose:** Closes MP3 file and releases decoder resources
- **Side effects:** Closes file, deallocates libmad structures

### Format Query Methods
- **Signatures:** `bool IsSixteenBit()`, `bool IsStereo()`, `bool IsSigned()`, `int BytesPerFrame()`, `float Rate()`, `bool IsLittleEndian()`
- **Purpose:** Return audio format metadata (all inline, no I/O)
- **Returns:** Format flags (hardcoded: 16-bit signed, endianness conditional on platform)

## Control Flow Notes
This decoder fits into the engine's audio streaming pipeline:
1. **Init:** Client calls `Open()` to load an MP3 file, queries format via `IsStereo()`, `Rate()`, etc.
2. **Update/Frame:** Client repeatedly calls `Decode()` in the audio thread to fill audio buffers
3. **Shutdown:** Client calls `Close()` to release resources; `Rewind()` available for looping playback

Decoding is pull-based: `Decode()` is called on demand, refilling `InputBuffer` as libmad consumes MP3 data.

## External Dependencies
- **libmad** (`<mad.h>`) ΓÇô MP3 frame parsing, synthesis
- **cseries.h** ΓÇô project type definitions and utilities
- **Decoder.h** ΓÇô `StreamDecoder` base class interface and factory
- **FileHandler.h** (via cseries) ΓÇô `FileSpecifier` and `OpenedFile` types
- **Conditional compilation:** `HAVE_MAD`, `ALEPHONE_LITTLE_ENDIAN`
