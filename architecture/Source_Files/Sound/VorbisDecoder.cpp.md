# Source_Files/Sound/VorbisDecoder.cpp

## File Purpose
Implements OGG/Vorbis audio file decoding for the Aleph One game engine. Acts as an SDL-backed decoder that reads Vorbis streams from files and extracts audio frames with metadata (sample rate, stereo/mono).

## Core Responsibilities
- Set up SDL-based I/O callbacks (read, seek, tell, close) for the Vorbis decoder
- Open and validate OGG/Vorbis files, extracting audio metadata (sample rate, channel count)
- Decode compressed audio frames into raw PCM buffers on demand
- Manage file seek/rewind operations
- Clean up Vorbis and file resources on close/destruction

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `VorbisDecoder` | class | Wraps libvorbisfile; implements `StreamDecoder` interface for audio streaming |
| `OggVorbis_File` | opaque struct (from libvorbisfile) | Handle to an open Vorbis stream |
| `ov_callbacks` | struct (from libvorbisfile) | Function pointers for file I/O (read, seek, tell, close) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sdl_read_func` | function pointer | static/file | Wraps SDL_RWread for Vorbis read callbacks |
| `sdl_seek_func` | function pointer | static/file | Wraps SDL_RWseek for Vorbis seek callbacks |
| `sdl_tell_func` | function pointer | static/file | Wraps SDL_RWtell for Vorbis tell callbacks |
| `sdl_close_func` | function pointer | static/file | Wraps SDL_RWclose for Vorbis close callbacks |
| `callbacks` | `ov_callbacks` | instance | Configured Vorbis callbacks (initialized in constructor) |
| `ov_file` | `OggVorbis_File` | instance | Active Vorbis file handle (valid after Open) |
| `stereo` | bool | instance | True if file has 2 channels, false if mono |
| `rate` | float | instance | Sample rate in Hz (e.g., 44100) |

## Key Functions / Methods

### VorbisDecoder() [constructor]
- **Purpose:** Initialize decoder state and configure SDL-based I/O callbacks
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates `callbacks` struct with SDL wrappers; sets `stereo=false`, `rate=0`
- **Calls:** No Vorbis or SDL calls
- **Notes:** Must be called before `Open()`; callbacks are set once and reused for all files

### Open(FileSpecifier& File)
- **Purpose:** Open and validate an OGG/Vorbis file, extract audio metadata
- **Inputs:** `File` ΓÇô FileSpecifier object describing the file to open
- **Outputs/Return:** `bool` ΓÇô true if file opened and validated successfully, false otherwise
- **Side effects:** Calls `File.Open(file)` to get an SDL_RWops handle; populates `ov_file`, `stereo`, `rate`; on failure, `ov_file` may be partially initialized
- **Calls:** `File.Open()`, `ov_test_callbacks()`, `ov_test_open()`, `ov_info()`
- **Notes:** Returns false if file is not a valid Vorbis stream or if metadata extraction fails. `ov_info()` may return null if stream is invalid.

### Decode(uint8\* buffer, int32 max_length)
- **Purpose:** Decode and buffer up to `max_length` bytes of raw PCM audio
- **Inputs:** `buffer` ΓÇô destination byte buffer; `max_length` ΓÇô max bytes to decode
- **Outputs/Return:** `int32` ΓÇô number of bytes actually decoded and written to buffer
- **Side effects:** Advances playback position in `ov_file`; writes PCM data to `buffer`; updates `current_section` via `ov_read()`
- **Calls:** `ov_read()`, `IsLittleEndian()` (endianness check)
- **Notes:** Returns 0 or fewer bytes on EOF or error. Endian-aware (little-endian=0, big-endian=1 flag). Decodes 16-bit signed PCM. Loop continues until buffer full or stream exhausted.

### Rewind()
- **Purpose:** Reset playback position to the beginning of the file
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Seeks `ov_file` to byte 0
- **Calls:** `ov_raw_seek()`
- **Notes:** Raw seek (byte-level); does not reset metadata

### Close()
- **Purpose:** Clean up Vorbis decoder resources
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes `ov_file` and underlying SDL_RWops handle via `ov_clear()`
- **Calls:** `ov_clear()`
- **Notes:** Safe to call if file was never opened or already closed

### ~VorbisDecoder() [destructor]
- **Purpose:** Destructor for VorbisDecoder
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `Close()`
- **Calls:** `Close()`

## Control Flow Notes
**Initialization ΓåÆ Decoding ΓåÆ Cleanup:** Typical usage is constructor ΓåÆ `Open()` ΓåÆ repeated `Decode()` calls (e.g., in an audio frame loop) ΓåÆ optional `Rewind()` ΓåÆ `Close()` (or implicit in destructor). The `Open()` step initializes `ov_file` and extracts metadata (`stereo`, `rate`). The `Decode()` method is intended for repeated calls within a playback/streaming loop, returning data until EOF.

## External Dependencies
- **SDL (Simple DirectMedia Layer):** `SDL_RWops`, `SDL_RWread()`, `SDL_RWseek()`, `SDL_RWtell()`, `SDL_RWclose()` ΓÇô file I/O abstraction
- **libvorbisfile:** `OggVorbis_File`, `ov_test_callbacks()`, `ov_test_open()`, `ov_info()`, `ov_read()`, `ov_raw_seek()`, `ov_clear()`, `ov_callbacks` ΓÇô Vorbis decoding
- **Local headers:** `"VorbisDecoder.h"` (class definition), `"cseries.h"`, `"Decoder.h"` (base class `StreamDecoder`)
- **Conditional compilation:** `#ifdef HAVE_VORBISFILE` (entire file only compiled if Vorbis support is enabled); `#ifdef ALEPHONE_LITTLE_ENDIAN` (endianness query in header)
