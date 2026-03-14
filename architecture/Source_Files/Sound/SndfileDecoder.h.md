# Source_Files/Sound/SndfileDecoder.h

## File Purpose
Declares `SndfileDecoder`, a concrete decoder for audio files using the libsndfile library. Inherits from the abstract `Decoder` base class to provide streaming PCM decoding for the game engine's audio system. Only compiled when libsndfile support is available (guarded by `#ifdef HAVE_SNDFILE`).

## Core Responsibilities
- Decode audio files (WAV, FLAC, OGG Vorbis, etc.) supported by libsndfile into raw PCM buffers
- Manage the lifecycle of a libsndfile decoder handle (`SNDFILE*`)
- Report audio format metadata (bit depth, channels, sample rate, endianness, total frame count)
- Support sequential decoding and stream rewinding
- Maintain consistency with the abstract decoder interface for polymorphic usage

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SndfileDecoder` | class | Concrete decoder wrapping libsndfile functionality |
| `SNDFILE` | opaque pointer | Handle to a libsndfile decoder instance (from sndfile.h) |
| `SF_INFO` | struct | Libsndfile metadata structure (channels, sample rate, frames, etc.) |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- **Signature:** `bool Open(FileSpecifier& File)`
- **Purpose:** Open and initialize a sound file for decoding
- **Inputs:** `File` ΓÇö file specifier pointing to the audio file
- **Outputs/Return:** `true` if successful, `false` otherwise
- **Side effects:** Allocates `sndfile` handle; populates `sfinfo` with file metadata
- **Calls:** Calls `sndfile.h` functions (not visible in this file)
- **Notes:** Abstract method from `Decoder`; must be called before `Decode()`

### Decode
- **Signature:** `int32 Decode(uint8* buffer, int32 max_length)`
- **Purpose:** Decode audio frames from the current position into a PCM buffer
- **Inputs:** `buffer` ΓÇö destination byte buffer; `max_length` ΓÇö maximum bytes to write
- **Outputs/Return:** Number of bytes decoded (0 at end-of-file)
- **Side effects:** Advances internal file pointer
- **Calls:** Libsndfile read functions (not visible in this file)
- **Notes:** Abstract method from `StreamDecoder`; returns bytes written, not frame count

### Rewind
- **Signature:** `void Rewind()`
- **Purpose:** Reset the decoder to the beginning of the file
- **Side effects:** Resets libsndfile's internal read pointer
- **Notes:** Abstract method from `StreamDecoder`

### Close
- **Signature:** `void Close()`
- **Purpose:** Close the sound file and release the decoder handle
- **Side effects:** Deallocates `sndfile` handle
- **Notes:** Abstract method from `StreamDecoder`; safe to call multiple times

### Property accessors
- **`IsSixteenBit()`** ΓåÆ `true` ΓÇö libsndfile output is always 16-bit PCM
- **`IsStereo()`** ΓåÆ checks `sfinfo.channels == 2`
- **`IsSigned()`** ΓåÆ `true` ΓÇö libsndfile outputs signed PCM
- **`BytesPerFrame()`** ΓåÆ `2 * (mono ? 1 : 2)` ΓÇö bytes per frame (16-bit ├ù 1 or 2 channels)
- **`Rate()`** ΓåÆ `(float) sfinfo.samplerate` ΓÇö sample rate in Hz
- **`IsLittleEndian()`** ΓåÆ conditional on `ALEPHONE_LITTLE_ENDIAN` build flag
- **`Frames()`** ΓåÆ `sfinfo.frames` ΓÇö total number of audio frames

## Control Flow Notes
This decoder is instantiated polymorphically via `Decoder::Get()` or `StreamDecoder::Get()` (defined in base class). Typical usage flow: `Open()` ΓåÆ loop `Decode()` ΓåÆ `Close()`. Rewind support allows replaying audio. Frame count is available before decoding begins.

## External Dependencies
- **`Decoder.h`** ΓÇö abstract base class `Decoder` and `StreamDecoder`
- **`sndfile.h`** ΓÇö external libsndfile library (provides `SNDFILE`, `SF_INFO`, and decode functions)
- **`FileHandler.h`** ΓÇö defines `FileSpecifier` (via Decoder.h)
- **`cseries.h`** ΓÇö defines standard types like `uint8`, `int32` (via Decoder.h)
