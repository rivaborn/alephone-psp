# Source_Files/Sound/SoundFile.cpp

## File Purpose
Implements sound file loading and management for the Aleph One game engine. Handles parsing System 7 sound formats, loading sound definitions and permutations, and supports dynamic custom sound addition via external decoders.

## Core Responsibilities
- Parse and unpack System 7 sound headers (standard 22-byte and extended 64-byte formats)
- Load sound sample data from files into memory
- Manage hierarchical sound definitions (sources ΓåÆ sounds ΓåÆ permutations)
- Support dynamic custom sound definitions and external format loading
- Validate sound file structure and metadata (magic tag, version, sample rates, loop points)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| SoundHeader | struct | Single audio sample with format metadata (bit depth, channels, rate, loop points) |
| SoundDefinition | struct | Sound definition with multiple permutations, pitch range, play chance, offsets |
| SoundFile | class | Root sound resource container; manages 2D array of definitions (sources ├ù sounds) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TryLoadingExternal` | function | static | Helper to load external audio files via Decoder system |

## Key Functions / Methods

### SoundHeader::UnpackStandardSystem7Header
- **Signature:** `bool UnpackStandardSystem7Header(AIStreamBE &header)`
- **Purpose:** Parse 22-byte System 7 standard sound header format
- **Inputs:** Big-endian stream positioned at header start
- **Outputs/Return:** `true` if valid, `false` on parse error
- **Side effects:** Populates `bytes_per_frame`, `rate`, `loop_start`, `loop_end`, `length`
- **Calls:** (none visible)
- **Notes:** Sets `mono`, `8-bit unsigned` defaults; uses try/catch for safety

### SoundHeader::UnpackExtendedSystem7Header
- **Signature:** `bool UnpackExtendedSystem7Header(AIStreamBE &header)`
- **Purpose:** Parse 64-byte extended header; handles stereo, 16-bit, AIFF-C compression
- **Inputs:** Big-endian stream, header buffer
- **Outputs/Return:** `true` if valid; `false` if format incompatible
- **Side effects:** Validates compression format; converts loop offsets if needed (logs warning); sets channel/bit-depth flags
- **Calls:** `logWarning3` (if loop offset mismatch detected)
- **Notes:** Detects AIFF-C compression via format code and codec ID; validates signed/unsigned bit; accounts for frame-vs-byte offset confusion in loop points

### SoundHeader::Load (const uint8* variant)
- **Signature:** `bool Load(const uint8* data)`
- **Purpose:** Parse header from memory; does not copy audio data (points to external buffer)
- **Inputs:** Raw byte pointer to sound data
- **Outputs/Return:** `true` if header valid
- **Side effects:** Sets `this->data` pointer; does not populate `stored_data`
- **Calls:** `UnpackStandardSystem7Header`, `UnpackExtendedSystem7Header`
- **Notes:** Dispatches on header type byte at offset 20

### SoundHeader::Load (OpenedFile& variant)
- **Signature:** `bool Load(OpenedFile &SoundFile)`
- **Purpose:** Load header and audio samples from file into owned storage
- **Inputs:** Open file handle positioned at sound header start
- **Outputs/Return:** `true` if successful
- **Side effects:** Reads header and sample data from file; populates `stored_data` vector
- **Calls:** `UnpackStandardSystem7Header`, `UnpackExtendedSystem7Header`
- **Notes:** Owns data after load; uses exception safety pattern with try/catch

### SoundDefinition::Unpack
- **Signature:** `bool Unpack(OpenedFile &SoundFile)`
- **Purpose:** Read 64-byte sound definition metadata from file
- **Inputs:** Open file handle
- **Outputs/Return:** `true` if valid
- **Side effects:** Populates `sound_code`, `behavior_index`, `flags`, pitch, permutation counts, offsets array
- **Calls:** (none visible)
- **Notes:** Reads up to 5 sound offsets; ignores trailing reserved fields

### SoundDefinition::Load
- **Signature:** `bool Load(OpenedFile &SoundFile, bool LoadPermutations)`
- **Purpose:** Load actual sound sample data for this definition's permutations
- **Inputs:** File handle; flag to load all permutations or just first
- **Outputs/Return:** `true` if all samples loaded
- **Side effects:** Populates `sounds` vector; clears on error
- **Calls:** `SoundHeader::Load` (file variant)
- **Notes:** Seeks to group offset + permutation offset; lazy-loads from file

### SoundFile::Open
- **Signature:** `bool Open(FileSpecifier& SoundFileSpec)`
- **Purpose:** Load entire sound resource file
- **Inputs:** File specifier pointing to sound file
- **Outputs/Return:** `true` if valid sound file
- **Side effects:** Validates version/tag; resizes `sound_definitions` 2D array; keeps file open in `opened_sound_file`
- **Calls:** `SoundDefinition::Unpack` (nested loops over sources ├ù sounds)
- **Notes:** Normalizes sound_count==0 case (single source); early exit on validation failure

### SoundFile::Close
- **Signature:** `void Close()`
- **Purpose:** Unload all sound definitions
- **Inputs:** (none)
- **Outputs/Return:** (none)
- **Side effects:** Clears all sound definitions; closes file handle (via auto_ptr cleanup)

### SoundFile::NewCustomSoundDefinition
- **Signature:** `int NewCustomSoundDefinition()`
- **Purpose:** Allocate new sound definition for custom (user-added) sounds
- **Inputs:** (none)
- **Outputs/Return:** New sound index
- **Side effects:** Increments `sound_count`; adds new SoundDefinition to each source group; assigns sound code (19000+)
- **Calls:** `machine_tick_count`
- **Notes:** Asserts that all source groups have matching size after insertion

### TryLoadingExternal (static)
- **Signature:** `static bool TryLoadingExternal(SoundHeader& hdr, const char* path)`
- **Purpose:** Load external audio file (MP3, OGG, etc.) via Decoder
- **Inputs:** SoundHeader to populate; file path string
- **Outputs/Return:** `true` if decode succeeded
- **Side effects:** Allocates and populates SoundHeader; calls `hdr.Load(length)` to allocate buffer; fills format flags from decoder
- **Calls:** `Decoder::Get`, decoder methods (`Frames()`, `BytesPerFrame()`, `Decode()`, etc.)
- **Notes:** Sets loop to [0, 0]; uses FIXED_ONE scale for rate; includes explanatory comment about weirdness workarounds

### SoundFile::AddCustomSoundSlot
- **Signature:** `bool AddCustomSoundSlot(int index, const char* file)`
- **Purpose:** Add external sound file as a permutation to existing sound definition
- **Inputs:** Custom sound index; file path to external format
- **Outputs/Return:** `true` if loaded successfully
- **Side effects:** Appends SoundHeader to `sounds` vector (up to 5 permutations); increments permutation count
- **Calls:** `TryLoadingExternal`
- **Notes:** Index must be in custom range; silently skips if permutation limit reached

### SoundFile::GetSoundDefinition
- **Signature:** `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- **Purpose:** Retrieve pointer to sound definition
- **Inputs:** Source and sound indices
- **Outputs/Return:** Pointer to definition, or `nullptr` if out of bounds
- **Side effects:** (none)
- **Calls:** (none)

**Trivial helpers:** `SoundHeader::Clear()`, `SoundHeader::Data()`, `SoundHeader::Length()`, `SoundDefinition::LoadedSize()` ΓÇö constructor, accessors, and teardown.

## Control Flow Notes

**Startup:** `Open()` loads file header and all sound definition metadata; file handle is retained for lazy loading. Audio samples are loaded on-demand via `SoundDefinition::Load()`.

**Runtime:** Game queries `GetSoundDefinition()`, then calls `Load()` on demand. Custom sounds are allocated via `NewCustomSoundDefinition()` and populated via `AddCustomSoundSlot()` with external files.

**Cleanup:** `Close()` releases all definitions and closes the file.

## External Dependencies

- **AStream.h** ΓÇö `AIStreamBE` (big-endian binary stream reader)
- **FileHandler.h** ΓÇö `OpenedFile`, `FileSpecifier` (file I/O abstraction)
- **Decoder.h** ΓÇö `Decoder` (external sound format decoder; supports multiple codecs)
- **Logging.h** ΓÇö `logWarning3` (warning logging)
- **csmisc.h** ΓÇö `machine_tick_count()` (platform tick counter), `FIXED_ONE` (fixed-point unity)
- **Standard library** ΓÇö `<vector>`, `<memory>` (auto_ptr), `<assert.h>`

**Defined elsewhere:** `FOUR_CHARS_TO_INT` macro, `MAXIMUM_PERMUTATIONS_PER_SOUND` constant.
