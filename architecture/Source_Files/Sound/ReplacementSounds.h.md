# Source_Files/Sound/ReplacementSounds.h

## File Purpose
Provides a singleton-managed system for loading and retrieving external sound files as replacements for built-in game sounds. Implements hash-based lookup for efficient runtime sound retrieval during gameplay based on sound index and permutation slot.

## Core Responsibilities
- Define `ExternalSoundHeader` to load external sound files from disk
- Manage a registry of sound replacements indexed by sound ID and permutation slot
- Provide O(1) hash-table lookup for real-time sound retrieval
- Singleton access pattern for global sound replacement management
- Support reset/clearing of all replacements

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| ExternalSoundHeader | class | Extends SoundHeader to load external sound files from FileSpecifier |
| SoundOptions | struct | Holds file path and loaded ExternalSoundHeader for a replacement sound |
| SoundOptionsEntry | struct | Wrapper pairing a SoundOptions with its Index/Slot identifiers |
| SoundReplacements | class | Singleton manager for storing and retrieving sound replacements via hash table |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | SoundReplacements* | static class member | Singleton instance |
| SOList | std::vector<SoundOptionsEntry> | private member | Dense storage of all replacement sound entries |
| SOHash | std::vector<int16> | private member | Hash table (size 256) mapping hash values to SOList indices |

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- Signature: `bool LoadExternal(FileSpecifier& File)`
- Purpose: Load and parse an external sound file from disk
- Inputs: FileSpecifier reference pointing to the sound file
- Outputs/Return: Boolean success flag
- Side effects: Populates inherited SoundHeader members (format, sample rate, loop points, audio data)
- Calls: Not visible in this file
- Notes: Implementation deferred to .cpp; overrides parent class loading mechanism

### SoundReplacements::instance
- Signature: `static inline SoundReplacements* instance()`
- Purpose: Access singleton instance with lazy initialization
- Inputs: None
- Outputs/Return: Pointer to static SoundReplacements instance
- Side effects: Allocates SoundReplacements on first call
- Calls: `new SoundReplacements`
- Notes: Classic lazy singleton; not thread-safe; inlined for minimal call overhead

### SoundReplacements::GetSoundOptions
- Signature: `SoundOptions* GetSoundOptions(short Index, short Slot)`
- Purpose: Retrieve replacement sound for a given audio event and variant
- Inputs: Index (sound identifier), Slot (permutation/variant number)
- Outputs/Return: Pointer to SoundOptions, nullptr if not registered
- Side effects: None (read-only)
- Calls: HashFunc to compute hash value
- Notes: Critical path during sound playback; hash-table lookup avoids linear search

### SoundReplacements::Add
- Signature: `void Add(const SoundOptions& Data, short Index, short Slot)`
- Purpose: Register a new sound replacement in the hash table
- Inputs: SoundOptions (file + loaded data), Index, Slot
- Outputs/Return: Void
- Side effects: Appends to SOList; updates SOHash entry
- Calls: HashFunc to compute hash location
- Notes: No explicit collision handling or duplicate detection visible

### SoundOptions::HashFunc
- Signature: `static inline uint8 HashFunc(short Index, short Slot)`
- Purpose: Compute hash index for Index/Slot pair
- Inputs: Index, Slot
- Outputs/Return: uint8 (range 0ΓÇô255)
- Side effects: None
- Calls: None
- Notes: Uses `Index ^ (Slot << 4)` to distribute permutations of the same sound index across the hash table; avoids collisions for common access patterns

## Control Flow Notes
Fits into sound subsystem initialization: external sound files are loaded during startup/MML parsing via `Add()`, then looked up at runtime via `GetSoundOptions()` during sound playback. The singleton is likely initialized once and accessed repeatedly during frames where audio events are triggered.

## External Dependencies
- `<string>` ΓÇö for file paths
- `SoundFile.h` ΓÇö provides `SoundHeader` base class and `FileSpecifier` type
- Aleph One engine framework (license headers, file I/O abstractions)

## Notes
- **Potential issue:** `Reset()` clears SOList but not SOHash, leaving stale hash table entries.
- **Design assumption:** HashSize of 256 is static and decoupled from actual sound count; may have poor distribution if sounds cluster around certain indices.
- **Thread safety:** Singleton and hash table are not protected; unsafe for concurrent access.
