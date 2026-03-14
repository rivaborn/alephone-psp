# Source_Files/Sound/Decoder.h

## File Purpose
Defines abstract base classes for audio decoding with factory pattern support. `StreamDecoder` provides streaming decode operations and audio format queries; `Decoder` extends it to add frame-count capability for full-file operations.

## Core Responsibilities
- Provide abstract interface for opening, decoding, and closing audio streams
- Expose audio format metadata (bit depth, sample rate, channel count, endianness, signedness)
- Support factory pattern for instantiating appropriate decoder implementations based on file type
- Separate streaming decoders from full-file decoders with frame information
- Allow repeated decoding and rewinding of audio data

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| StreamDecoder | abstract class | Base interface for audio stream decoding and format queries |
| Decoder | abstract class | Extended interface adding total frame count capability |

## Global / File-Static State
None

## Key Functions / Methods

### StreamDecoder::Open
- Signature: `virtual bool Open(FileSpecifier &File) = 0`
- Purpose: Open an audio file for decoding
- Inputs: `FileSpecifier &File` ΓÇô file to open
- Outputs/Return: `bool` ΓÇô success/failure
- Side effects: Initializes decoder state, opens file handle
- Notes: Pure virtual; responsibility of subclass to implement platform/format-specific logic

### StreamDecoder::Decode
- Signature: `virtual int32 Decode(uint8* buffer, int32 max_length) = 0`
- Purpose: Decode audio data from current stream position into caller's buffer
- Inputs: `uint8* buffer` ΓÇô output buffer, `int32 max_length` ΓÇô max bytes to fill
- Outputs/Return: `int32` ΓÇô bytes written to buffer (typically Γëñ max_length)
- Side effects: Advances stream position, modifies buffer contents
- Notes: Pure virtual; typical implementation fills buffer incrementally; return value indicates EOF when < max_length

### StreamDecoder::Rewind / Close
- Signature: `virtual void Rewind() = 0`, `virtual void Close() = 0`
- Purpose: Reset stream position to start; release file handle and resources
- Inputs: None
- Side effects: Rewind advances stream pointer only; Close releases resources
- Notes: Both pure virtual; Close called before destruction or when done decoding

### Format Query Methods
- Signatures: `IsSixteenBit()`, `IsStereo()`, `IsSigned()`, `IsLittleEndian()`, `BytesPerFrame()`, `Rate()`
- Purpose: Retrieve audio format properties for buffer allocation and interpretation
- Outputs/Return: `bool` (first five); `int` (BytesPerFrame); `float` (Rate in Hz)
- Notes: All pure virtual; called after Open to determine decode behavior

### StreamDecoder::Get (factory)
- Signature: `static StreamDecoder* Get(FileSpecifier &File)`
- Purpose: Factory method instantiating appropriate streaming decoder based on file type
- Inputs: `FileSpecifier &File` ΓÇô file to decode
- Outputs/Return: `StreamDecoder*` ΓÇô pointer to concrete decoder, or null if unsupported format
- Side effects: Dynamic allocation of subclass instance
- Notes: Implementation elsewhere; nullptr return indicates unknown/unsupported file type

### Decoder::Frames / Decoder::Get
- Signatures: `virtual int32 Frames() = 0`, `static Decoder* Get(FileSpecifier &File)`
- Purpose: Frames() returns total audio frames in file; Get() factory for full-file decoders
- Outputs/Return: `int32` frame count; `Decoder*` pointer or null
- Notes: Decoder subclass supports seeking/duration queries; Get() can return null

## Control Flow Notes
Typical usage pattern:
1. Call `Get()` with a FileSpecifier to instantiate appropriate decoder
2. Call `Open()` to load/initialize the audio file
3. Query format methods (`IsStereo()`, `Rate()`, etc.) to determine buffer size and interpretation
4. Call `Decode()` repeatedly to fill buffers with audio data
5. Call `Rewind()` to restart if needed, or `Close()` to release resources

For operations requiring frame-level information (seeking, duration), use `Decoder` subclass instead of `StreamDecoder`.

## External Dependencies
- `cseries.h` ΓÇô engine type definitions (`int32`, `uint8`, `bool`)
- `FileHandler.h` ΓÇô `FileSpecifier` class for file abstraction
- Concrete decoder implementations not defined in this header (e.g., WAV, Ogg, MIDI decoders)
