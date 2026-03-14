# Source_Files/Sound/BasicIFFDecoder.h

## File Purpose
Decoder class for uncompressed AIFF and WAV audio files. Inherits from the `Decoder` abstract base class and implements frame-based streaming decoding with format metadata (bit depth, channels, sample rate, endianness).

## Core Responsibilities
- Open and validate uncompressed AIFF/WAV files
- Decode audio frames into a provided buffer
- Track playback position and provide frame count
- Expose audio format properties (mono/stereo, 8/16-bit, signed/unsigned, sample rate)
- Manage file handle and buffer offset during streaming

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| BasicIFFDecoder | class | Concrete decoder for IFF-format audio (AIFF/WAV) |

## Global / File-Static State
None.

## Key Functions / Methods

### Open
- Signature: `bool Open(FileSpecifier &File)`
- Purpose: Open and parse an IFF audio file, reading format metadata
- Inputs: File reference
- Outputs/Return: `true` if file opened and format parsed successfully
- Side effects: Populates `sixteen_bit`, `stereo`, `signed_8bit`, `bytes_per_frame`, `rate`, `little_endian`, `length`, `data_offset`; opens `file` handle
- Calls: (implementation-dependent; reads IFF header chunks)
- Notes: Must validate FORM chunk and locate audio data chunk

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Read and return next chunk of decoded audio samples into provided buffer
- Inputs: `buffer` (destination), `max_length` (max bytes to decode)
- Outputs/Return: Number of bytes decoded (Γëñ `max_length`); 0 at end of stream
- Side effects: Advances file read position (`data_offset`)
- Calls: (file I/O from `OpenedFile`)
- Notes: Respects `bytes_per_frame` for alignment; returns 0 when `Done()`

### IsSixteenBit, IsStereo, IsSigned, Rate, IsLittleEndian
- Purpose: Query audio format properties
- Return: Format metadata populated during `Open`

### BytesPerFrame
- Purpose: Bytes per sample frame (channels ├ù bytes-per-sample)
- Return: Calculated/cached value for buffer alignment

### Frames
- Signature: `int32 Frames()`
- Purpose: Total frame count in file
- Return: `length / bytes_per_frame`
- Notes: Only valid after `Open()` succeeds

### Rewind, Close, Done
- Purpose: Reposition to start; close file handle; check end-of-stream
- Notes: `Done()` implicitly checks if all frames have been decoded

## Control Flow Notes
Part of the audio subsystem's decoder pipeline. Called during music/sound loading and streaming playback. The decoder is selected by `Decoder::Get()` based on file format, then `Open()` reads metadata, `Decode()` streams frames on demand, and `Close()` releases resources. Position is tracked implicitly via `data_offset` and compared against `length` to determine `Done()`.

## External Dependencies
- `Decoder.h` ΓÇô parent class `Decoder : public StreamDecoder`
- `FileHandler.h` ΓÇô `FileSpecifier`, `OpenedFile` (defined elsewhere)
- `cseries.h` ΓÇô basic types (`uint8`, `int32`, `float`; defined elsewhere)
