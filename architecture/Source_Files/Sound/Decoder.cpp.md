# Source_Files/Sound/Decoder.cpp

## File Purpose
Factory methods for instantiating audio format decoders based on available compile-time features and file format detection. Attempts decoders in priority order, returning the first one that successfully opens the given file.

## Core Responsibilities
- Static factory for StreamDecoder instances (supports SNDFILE, VORBIS, MAD, and BasicIFF fallback)
- Static factory for Decoder instances (frame-aware decoders; BasicIFF fallback)
- Format auto-detection via sequential decoder instantiation and Open() attempts
- Ownership transfer via auto_ptr::release() to caller
- Conditional compilation to select available audio codec support

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### StreamDecoder::Get
- **Signature:** `static StreamDecoder* StreamDecoder::Get(FileSpecifier& File)`
- **Purpose:** Factory method that tries available audio format decoders to find one capable of opening the file.
- **Inputs:** `FileSpecifier& File` ΓÇô audio file to open.
- **Outputs/Return:** `StreamDecoder*` ΓÇô pointer to successfully opened decoder instance, or `null` (0) if none succeed.
- **Side effects:** Allocates decoder objects via `new`; ownership transferred to caller via `auto_ptr::release()`.
- **Calls:** Decoder constructors (SndfileDecoder, BasicIFFDecoder, VorbisDecoder, MADDecoder) and `Open()` methods.
- **Notes:** Tries decoders in priority order: SNDFILE > (BasicIFF fallback if no SNDFILE) > VORBIS > MAD. Returns first successful open; others are discarded. Uses early-exit pattern with auto_ptr scoping.

### Decoder::Get
- **Signature:** `static Decoder* Decoder::Get(FileSpecifier& File)`
- **Purpose:** Factory method for frame-aware decoders (extends StreamDecoder with total frame count).
- **Inputs:** `FileSpecifier& File` ΓÇô audio file to open.
- **Outputs/Return:** `Decoder*` ΓÇô pointer to opened decoder, or `null` if none succeed.
- **Side effects:** Allocates decoder objects via `new`.
- **Calls:** SndfileDecoder and BasicIFFDecoder constructors and `Open()` methods.
- **Notes:** **Incomplete implementation** ΓÇô only tries SNDFILE or BasicIFF; no explicit `return` in #else branch (implies undefined return). Missing VORBIS and MAD support unlike `StreamDecoder::Get()`.

## Control Flow Notes
Entry points for audio resource initialization. Called when loading music or SFX files during game startup/runtime. Codec availability determined at compile time; runtime dispatch selects the first compatible decoder. Falls back to uncompressed IFF (AIFF/WAV) if licensed codecs are unavailable.

## External Dependencies
- **Includes:** Decoder.h, BasicIFFDecoder.h, MADDecoder.h, SndfileDecoder.h, VorbisDecoder.h, memory (std::auto_ptr)
- **Defined elsewhere:** FileSpecifier (file abstraction), all Decoder subclasses (BasicIFFDecoder, MADDecoder, SndfileDecoder, VorbisDecoder)
- **Build-time toggles:** HAVE_SNDFILE, HAVE_VORBISFILE, HAVE_MAD
