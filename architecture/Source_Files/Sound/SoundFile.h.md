# Source_Files/Sound/SoundFile.h

## File Purpose
Defines sound resource management classes for loading, storing, and accessing sound data in System 7 format. Part of the Aleph One game engine's audio subsystem, handling both built-in and custom sound definitions with pitch variation and looping support.

## Core Responsibilities
- Parse and load System 7 sound format from files and memory buffers
- Manage sound sample data storage with format metadata (bit depth, stereo, endianness)
- Organize sound definitions with multiple permutations (variants) per sound
- Provide central sound file interface for opening, loading, and retrieving sound definitions
- Support custom sound slots for runtime-added audio resources
- Cache opened sound files to optimize repeated access

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SoundHeader` | class | Container for decoded audio sample data with format info (bit depth, channels, sample rate, loop points) |
| `SoundDefinition` | class | Groups multiple sound permutations with playback parameters (pitch range, chance to play, behavior flags) |
| `SoundFile` | class | Manages file I/O and indexing of all sound definitions; caches opened sound file |

## Global / File-Static State
None.

## Key Functions / Methods

### SoundHeader::Load (from OpenedFile)
- Signature: `bool Load(OpenedFile &SoundFile)`
- Purpose: Parse System 7 sound format from an open file handle and populate header fields
- Inputs: Open file positioned at sound data
- Outputs/Return: `true` if parse succeeded
- Side effects: Reads from file; populates `sixteen_bit`, `stereo`, `signed_8bit`, `rate`, `loop_start`, `loop_end`
- Calls: `UnpackStandardSystem7Header()`, `UnpackExtendedSystem7Header()` (via AIStreamBE)
- Notes: Handles both standard and extended System 7 headers; endianness determined during parse

### SoundHeader::Load (from memory buffer)
- Signature: `bool Load(const uint8* data)`
- Purpose: Parse System 7 sound from a raw byte buffer without storing it
- Inputs: Pointer to sound data in memory
- Outputs/Return: `true` if parse succeeded
- Side effects: Populates header fields; does NOT allocate `stored_data`
- Notes: Used for in-place parsing; metadata only

### SoundHeader::Load (allocation)
- Signature: `uint8* Load(int32 length)`
- Purpose: Allocate storage for decoded audio samples
- Inputs: Number of bytes to allocate
- Outputs/Return: Pointer to allocated buffer for caller to fill
- Side effects: Clears existing data; resizes `stored_data` vector
- Notes: Caller responsible for decoding samples into returned buffer

### SoundFile::Open
- Signature: `bool Open(FileSpecifier &SoundFile)`
- Purpose: Open a sound resource file and parse its header directory
- Inputs: FileSpecifier pointing to .snd or equivalent resource file
- Outputs/Return: `true` if file opened and header parsed
- Side effects: Populates `version`, `tag`, `source_count`, `sound_count`, `sound_definitions` vector
- Calls: Reads 260-byte header; constructs sound definition objects from file
- Notes: Stores opened file in `opened_sound_file` auto_ptr for lazy loading

### SoundFile::GetSoundDefinition
- Signature: `SoundDefinition* GetSoundDefinition(int source, int sound_index)`
- Purpose: Retrieve a sound definition by source index and sound index
- Inputs: Source ID (row), sound index (column in 2D vector)
- Outputs/Return: Pointer to SoundDefinition, or nullptr if out of bounds
- Notes: Does not load sample data; `Load()` must be called separately

### SoundFile::Load
- Signature: `void Load(int source, int sound_index)`
- Purpose: Load sound sample data for a specific sound definition from file
- Inputs: Source and sound indices
- Side effects: Reads from `opened_sound_file`; populates `SoundHeader::stored_data` in the target SoundDefinition
- Notes: Lazy loading; only decodes on demand

### SoundFile::NewCustomSoundDefinition
- Signature: `int NewCustomSoundDefinition()`
- Purpose: Create a new empty sound slot for runtime-added custom audio
- Outputs/Return: New sound index; always returns a valid index
- Notes: Sound data is added later via `AddCustomSoundSlot()`

### SoundFile::AddCustomSoundSlot
- Signature: `bool AddCustomSoundSlot(int index, const char* file)`
- Purpose: Load custom audio from external file into a previously allocated sound slot
- Inputs: Slot index (from `NewCustomSoundDefinition()`), file path
- Outputs/Return: `true` if file loaded successfully
- Notes: Allows user-provided sounds to be loaded at runtime

### SoundFile::UnloadCustomSounds
- Signature: `void UnloadCustomSounds()`
- Purpose: Deallocate all custom sound data
- Side effects: Clears `stored_data` in all sound definitions
- Notes: Does not unload built-in sounds, only user-added ones

## Control Flow Notes
Part of the **resource initialization pipeline**. Called during game startup to load all sound definitions from the sound resource file. Individual sound data is **lazily loaded** on first access via `Load()`. The two-level structure (`SoundFile` ΓåÆ `SoundDefinition` ΓåÆ `SoundHeader`) allows efficient indexing of thousands of sounds while keeping memory usage low until a sound is actually played.

## External Dependencies
- `AStream.h` ΓÇö `AIStreamBE` for big-endian binary deserialization (System 7 format is big-endian)
- `FileHandler.h` ΓÇö `OpenedFile`, `FileSpecifier` for file I/O abstraction
- `<memory>` ΓÇö `std::auto_ptr` for RAII file handle
- `<vector>` ΓÇö `std::vector` for dynamic sound and permutation arrays
- **Defined elsewhere:** `_fixed` type (fixed-point arithmetic for pitch); `OpenedFile`, `FileSpecifier` (file I/O); stream operators in `AIStreamBE`
