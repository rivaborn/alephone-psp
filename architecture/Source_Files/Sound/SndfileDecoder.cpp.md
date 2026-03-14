# Source_Files/Sound/SndfileDecoder.cpp

## File Purpose
Implements a sound decoder that wraps libsndfile, enabling the engine to read and decode audio samples from various sound file formats (WAV, FLAC, etc.). Provides a simple streaming interface for decoding audio data in chunks and seeking within files.

## Core Responsibilities
- Open sound files and validate format metadata via libsndfile
- Decode audio samples into a buffer for playback
- Rewind/seek to the beginning of a sound file
- Manage lifetime of libsndfile file handles (open/close)
- Safely handle resource cleanup in constructor and destructor

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SNDFILE` | opaque pointer (libsndfile) | Encapsulates open file state and read position |
| `SF_INFO` | struct (libsndfile) | Stores sound file metadata (format, channels, sample rate, frame count) |

## Global / File-Static State
None.

## Key Functions / Methods

### Constructor
- Signature: `SndfileDecoder()`
- Purpose: Initialize the decoder with no active file
- Inputs: None
- Outputs/Return: N/A
- Side effects: Sets `sndfile` to null pointer
- Calls: None
- Notes: Simple initialization; no validation

### Destructor
- Signature: `~SndfileDecoder()`
- Purpose: Ensure file handle is properly closed on object destruction
- Inputs: None
- Outputs/Return: N/A
- Side effects: Calls `Close()`, releasing libsndfile resources
- Calls: `Close()`
- Notes: Prevents resource leaks

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open a sound file and read its metadata
- Inputs: `File` ΓÇô file specifier identifying the sound file to open
- Outputs/Return: `true` if file opened successfully; `false` otherwise
- Side effects: Closes any previously open file; initializes `sfinfo` and `sndfile`
- Calls: `sf_close()` (libsndfile), `sf_open()` (libsndfile), `File.GetPath()`
- Notes: Clears `sfinfo.format` before opening to reset state; `SFM_READ` indicates read-only mode

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Read and decode audio samples from the currently open file into a buffer
- Inputs: `buffer` ΓÇô output buffer; `max_length` ΓÇô buffer size in bytes
- Outputs/Return: Number of bytes decoded (read sample count ├ù 2)
- Side effects: Advances read position within the sound file
- Calls: `sf_read_short()` (libsndfile)
- Notes: Assumes 16-bit signed PCM; divides `max_length` by 2 because `sf_read_short()` works with 16-bit frames; returns 0 if no file is open

### Rewind
- Signature: `void Rewind()`
- Purpose: Seek to the beginning of the sound file
- Inputs: None
- Outputs/Return: N/A
- Side effects: Resets file read position to 0
- Calls: `sf_seek()` (libsndfile)
- Notes: No-op if no file is open

### Close
- Signature: `void Close()`
- Purpose: Close the currently open sound file and release libsndfile resources
- Inputs: None
- Outputs/Return: N/A
- Side effects: Closes file handle via libsndfile; sets `sndfile` to null
- Calls: `sf_close()` (libsndfile)
- Notes: Safe to call when no file is open; sets to null to prevent double-close

## Control Flow Notes
Typical usage: `Open()` ΓåÆ repeated `Decode()` calls (possibly with `Rewind()` for looping) ΓåÆ `Close()` or destruction. Part of a decoder hierarchy; engine likely instantiates this to stream audio from disk.

## External Dependencies
- `sndfile.h` ΓÇô libsndfile C API (file I/O, format detection, PCM decoding)
- `Decoder.h` ΓÇô base class defining the audio decoder interface
- `FileSpecifier` ΓÇô engine's file abstraction (GetPath method used)
