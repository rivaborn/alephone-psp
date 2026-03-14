# Source_Files/Sound/BasicIFFDecoder.cpp

## File Purpose
Decodes uncompressed WAV and AIFF audio files for playback. Extracts audio metadata (sample rate, bit depth, channels, endianness) from file headers and provides frame-by-frame audio data reading with playback controls (rewind, seek, done detection).

## Core Responsibilities
- Parse AIFF and WAV file headers to extract audio format information
- Detect and handle big-endian (AIFF) and little-endian (WAV) byte ordering
- Extract audio metadata: sample rate, channel count, bit depth, frame size
- Read and return raw audio frames to decoder consumer
- Manage file position during playback (current position, data offset, total length)
- Support playback control: rewind to start, detect end-of-file, close file handle

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `BasicIFFDecoder` | class | Audio format decoder; inherits from `Decoder` base class |

## Global / File-Static State
None.

## Key Functions / Methods

### BasicIFFDecoder (constructor)
- Signature: `BasicIFFDecoder()`
- Purpose: Initialize all member variables to default values (false/0)
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None
- Notes: Sets all boolean flags to false, integers to 0

### Open
- Signature: `bool Open(FileSpecifier& File)`
- Purpose: Open, parse, and validate either an AIFF or WAV file; extract all audio metadata
- Inputs: `FileSpecifier& File` ΓÇö file to open
- Outputs/Return: `bool` ΓÇö true if successfully parsed; false if invalid format or missing required chunks
- Side effects: Stores file handle in member `file`; reads and caches metadata (sample rate, bit depth, stereo, endianness, data offset, length)
- Calls: 
  - `File.Open()` ΓåÆ opens file
  - `file.GetRWops()` ΓåÆ gets SDL RWops handle for binary reading
  - `SDL_ReadBE32/16()`, `SDL_ReadLE32/16()`, `SDL_RWseek()`, `SDL_RWtell()` ΓåÆ SDL I/O functions
- Notes: 
  - AIFF chunk parsing: reads COMM (common) chunk for format and SSND (sound data) chunk for data offset
  - WAV chunk parsing: reads fmt (format) chunk and data chunk
  - AIFF sample rate is stored as 80-bit IEEE float; decoder guesses rate from first 4 bytes only (44100, 22050, or 11025 Hz)
  - AIFF always signed 8-bit; WAV always unsigned (for 8-bit)
  - Validates format (PCM for WAV, returns false if not)
  - Chunk sizes are padded to even bytes; skips padding when seeking

### Decode
- Signature: `int32 Decode(uint8* buffer, int32 max_length)`
- Purpose: Read up to `max_length` bytes of audio data from current file position into buffer
- Inputs: `uint8* buffer` ΓÇö output buffer; `int32 max_length` ΓÇö max bytes to read
- Outputs/Return: `int32` ΓÇö number of bytes read; 0 if file not open or read fails
- Side effects: Advances file read position
- Calls: `file.IsOpen()`, `file.GetPosition()`, `file.Read()`
- Notes: Clamps read length to remaining audio data (does not read past `data_offset + length`)

### Done
- Signature: `bool Done()`
- Purpose: Check if playback has reached end of audio data
- Inputs: None
- Outputs/Return: `bool` ΓÇö true if file position >= data_offset + length or file is closed
- Side effects: None
- Calls: `file.IsOpen()`, `file.GetPosition()`

### Rewind
- Signature: `void Rewind()`
- Purpose: Seek file position back to start of audio data
- Inputs: None
- Outputs/Return: None
- Side effects: Moves file read position to `data_offset`
- Calls: `file.IsOpen()`, `file.SetPosition()`

### Close
- Signature: `void Close()`
- Purpose: Close file and clear state
- Inputs: None
- Outputs/Return: None
- Side effects: Sets `length = 0`; closes file handle if open
- Calls: `file.IsOpen()`, `file.Close()`

## Control Flow Notes
Typical usage: `Open()` ΓåÆ metadata extraction ΓåÆ repeated `Decode()` calls during playback ΓåÆ `Done()` to detect end ΓåÆ optional `Rewind()` for restart ΓåÆ `Close()`. Fits into audio playback pipeline after file selection and before DSP/mixing.

## External Dependencies
- **SDL_endian.h**: `SDL_ReadBE32`, `SDL_ReadLE32`, `SDL_ReadBE16`, `SDL_ReadLE16`, `SDL_RWops`, `SDL_RWtell`, `SDL_RWseek` (byte-order-aware binary I/O)
- **Decoder** (base class, defined elsewhere): Pure interface for audio decoders
- **FileSpecifier** (defined elsewhere): Abstraction over file I/O with position tracking
- **std::vector** (included but unused in this file)
