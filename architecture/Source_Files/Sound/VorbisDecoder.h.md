# Source_Files/Sound/VorbisDecoder.h

## File Purpose
Declares a `VorbisDecoder` class for decoding Ogg/Vorbis compressed audio files in the Aleph One game engine. Inherits from `StreamDecoder` base class to provide a unified interface for audio decoding.

## Core Responsibilities
- Implement Ogg/Vorbis file decoding via libvorbisfile library
- Open, decode, rewind, and close Vorbis audio files
- Report audio format metadata (bit depth, sample rate, stereo, endianness, signed/unsigned)
- Manage decoder state and file I/O callbacks

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `VorbisDecoder` | class | Concrete decoder implementation for Ogg/Vorbis format |
| `OggVorbis_File` | struct (external) | Libvorbisfile opaque handle for decoder state |
| `ov_callbacks` | struct (external) | Custom I/O callbacks for Vorbis decoder |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open and initialize a Vorbis file for decoding
- Inputs: `FileSpecifier` reference to the audio file
- Outputs/Return: `bool` success/failure
- Side effects: Populates `ov_file` and `callbacks`; may allocate decoder state

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Decode next chunk of Vorbis audio data into a PCM buffer
- Inputs: Output buffer pointer; maximum bytes to decode
- Outputs/Return: Number of bytes decoded (or Γëñ0 on end/error)
- Side effects: Updates decoder read position

### Rewind
- Signature: `void Rewind()`
- Purpose: Reset file playback to the beginning
- Side effects: Seeks Vorbis stream to start

### Close
- Signature: `void Close()`
- Purpose: Close the Vorbis file and clean up decoder resources
- Side effects: Deallocates decoder state

### Format Query Methods
- `IsSixteenBit()` ΓåÆ `true` (hardcoded)
- `IsStereo()` ΓåÆ stored in `stereo` field
- `IsSigned()` ΓåÆ `true` (hardcoded)
- `BytesPerFrame()` ΓåÆ `2 * (stereo ? 2 : 1)` (16-bit samples, 1ΓÇô2 channels)
- `Rate()` ΓåÆ stored in `rate` field (float sample rate)
- `IsLittleEndian()` ΓåÆ platform-dependent (checked at compile-time via `ALEPHONE_LITTLE_ENDIAN`)

## Control Flow Notes
This is a header-only interface declaration; implementation is in a corresponding .cpp file. Usage flow:
1. Instantiate `VorbisDecoder`
2. Call `Open()` during initialization/load phase
3. Call `Decode()` repeatedly in audio frame loop to stream PCM data
4. Call `Rewind()` on seek/restart
5. Call `Close()` during shutdown or file unload

## External Dependencies
- **cseries.h** ΓÇö core engine types and utilities
- **Decoder.h** ΓÇö abstract `StreamDecoder` base class
- **vorbis/vorbisfile.h** ΓÇö libvorbisfile library (conditional, `HAVE_VORBISFILE` guard)
- **FileHandler.h** ΓÇö `FileSpecifier` type (transitively via Decoder.h)

**Notes:** Compilation requires libvorbisfile dev headers and linking with `-lvorbisfile`. The conditional `#ifdef HAVE_VORBISFILE` allows the engine to build without Vorbis support if the library is unavailable.
