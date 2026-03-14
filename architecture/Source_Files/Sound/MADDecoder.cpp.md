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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| MADDecoder | class | Audio decoder inheriting from StreamDecoder; wraps libmad decoder state |
| mad_stream | struct (libmad) | Input buffer and frame boundary tracking for libmad |
| mad_frame | struct (libmad) | Decoded MPEG frame header and data |
| mad_synth | struct (libmad) | PCM synthesis state with decoded samples |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| MadFixedToSshort | function | static inline | Helper to convert libmad fixed-point (MAD_F_) to int16 |
| InputBuffer | uint8[INPUT_BUFFER_SIZE + MAD_BUFFER_GUARD] | member | Sliding input buffer for file reads |

## Key Functions / Methods

### MADDecoder() (constructor)
- **Signature:** `MADDecoder()`
- **Purpose:** Initialize the MP3 decoder with default member values and empty libmad structures.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `mad_stream_init()`, `mad_frame_init()`, `mad_synth_init()` to allocate/reset libmad state.
- **Calls:** libmad init functions
- **Notes:** Member vars (stereo, bytes_per_frame, rate, sample) left uninitialized until `Open()`.

### Open()
- **Signature:** `bool Open(FileSpecifier &File)`
- **Purpose:** Open an MP3 file and decode the first frame to extract audio metadata (stereo, sample rate).
- **Inputs:** FileSpecifier reference to the file to open
- **Outputs/Return:** `true` if first frame decoded successfully; `false` if file open or first decode fails
- **Side effects:** Calls `File.Open(file)`, sets member vars (stereo, bytes_per_frame, rate, sample = 0), sets file_done = false
- **Calls:** `File.Open()`, `DecodeFrame()`
- **Notes:** Must succeed before `Decode()` is called.

### Decode()
- **Signature:** `int32 Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decode and write PCM audio samples to output buffer up to max_length bytes.
- **Inputs:** Pointer to output buffer, max bytes to write
- **Outputs/Return:** Number of bytes actually written
- **Side effects:** Writes PCM data to buffer in system endianness; increments sample counter; calls `DecodeFrame()` when PCM buffer exhausted
- **Calls:** `MadFixedToSshort()`, `DecodeFrame()`
- **Notes:** Pads max_length down by 1ΓÇô3 bytes to avoid buffer overrun for mono/stereo alignment. Returns early if file exhausted.

### DecodeFrame()
- **Signature:** `bool DecodeFrame()`
- **Purpose:** Read and decode the next MP3 frame from the file into libmad structures.
- **Inputs:** None (uses member file, Stream, Frame, Synth, InputBuffer)
- **Outputs/Return:** `true` if frame decoded; `false` if error or EOF
- **Side effects:** Reads from file into InputBuffer, calls `mad_stream_buffer()`, `mad_frame_decode()`, `mad_synth_frame()`, sets sample = 0, sets file_done on guard-byte trigger
- **Calls:** `file.GetPosition()`, `file.GetLength()`, `file.Read()`, `memmove()`, `memset()`, libmad decode/synth functions
- **Notes:** Implements sliding input buffer to handle frame boundaries across reads. Appends MAD_BUFFER_GUARD zeros on EOF. Recursively retries on recoverable libmad errors (MAD_RECOVERABLE or MAD_ERROR_BUFLEN).

### Rewind()
- **Signature:** `void Rewind()`
- **Purpose:** Reset decoder state and file position for replay from the start.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls finish/init on libmad structures, seeks file to position 0, sets sample = 0 and file_done = false, calls `DecodeFrame()` to prime
- **Calls:** libmad finish/init functions, `file.SetPosition()`, `DecodeFrame()`
- **Notes:** Must be called before `Decode()` again after EOF.

### Close()
- **Signature:** `void Close()`
- **Purpose:** Clean up libmad state when done with the file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `mad_synth_finish()`, `mad_frame_finish()`, `mad_stream_finish()` to deallocate libmad internals
- **Calls:** libmad finish functions
- **Notes:** Called by destructor.

### MadFixedToSshort() (helper)
- **Signature:** `static inline int16 MadFixedToSshort(mad_fixed_t Fixed)`
- **Purpose:** Clamp and convert libmad fixed-point samples to signed 16-bit range.
- **Inputs:** Fixed-point sample value
- **Outputs/Return:** int16 clamped to [ΓêÆ32767, 32767]
- **Side effects:** None
- **Notes:** Clamps at ┬▒MAD_F_ONE boundary, shifts right by (MAD_F_FRACBITS ΓêÆ 15) bits.

## Control Flow Notes
**Initialization & Startup:**
Constructor initializes libmad state; `Open()` reads and decodes the first MP3 frame to detect format (sample rate, channels).

**Decoding Loop:**
`Decode()` is called repeatedly by the sound system. It extracts PCM samples from the internal Synth buffer one sample at a time, converting to host endianness. When the buffer is exhausted, `DecodeFrame()` reads more file data and decodes the next MP3 frame. The sliding input buffer allows frame boundaries to span file reads.

**Shutdown:**
`Close()` or `Rewind()` cleans up or reset libmad state. Destructor calls `Close()`.

## External Dependencies
- **Includes:** `<mad.h>` (libmad MP3 decoder library)
- **Inherits from:** `StreamDecoder` (defined elsewhere, likely abstract interface)
- **Uses:** `FileSpecifier` (file I/O abstraction, defined elsewhere)
- **Type macros:** `uint8`, `int16`, `int32`, `MAD_F_ONE`, `MAD_F_FRACBITS`, `SHRT_MAX` (platform/config defines)
- **Conditional compilation:** `#ifdef HAVE_MAD`, `#ifdef ALEPHONE_LITTLE_ENDIAN`
