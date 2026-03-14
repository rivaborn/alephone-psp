# Source_Files/Sound/ReplacementSounds.cpp

## File Purpose
Implements a singleton manager for external sound replacements in the Aleph One game engine. Loads audio files from disk, decodes them, and provides hash-table-based lookup for sound options indexed by audio type (Index) and variant (Slot).

## Core Responsibilities
- Load and decode external audio files via `Decoder` abstraction
- Manage a collection of sound options with efficient lookup by (Index, Slot) pair
- Implement a hash table with linear-search fallback for sound option retrieval
- Add or update sound options, avoiding duplicates by (Index, Slot)
- Maintain singleton instance of the replacement sound manager

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ExternalSoundHeader` | class | Extends `SoundHeader`; holds decoded audio data (bit depth, channels, sample rate, endianness, loop points) |
| `SoundOptions` | struct | Pairs a file path (`std::string File`) with decoded audio header; includes static hash function |
| `SoundOptionsEntry` | struct | Maps sound identity (Index, Slot) to associated `SoundOptions` data |
| `SoundReplacements` | class | Singleton manager; stores `SOList` (vector of entries) and `SOHash` (hash table indices) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `SoundReplacements::m_instance` | `SoundReplacements*` | static | Singleton instance pointer |

## Key Functions / Methods

### ExternalSoundHeader::LoadExternal
- **Signature:** `bool LoadExternal(FileSpecifier& File)`
- **Purpose:** Decode an external audio file and populate this header with audio metadata.
- **Inputs:** `File` ΓÇô file path/specifier to decode
- **Outputs/Return:** `true` if decode succeeded; `false` on error
- **Side effects:** Calls `Load(length)` (inherited from `SoundHeader`), allocates and populates audio buffer; sets `sixteen_bit`, `stereo`, `signed_8bit`, `bytes_per_frame`, `little_endian`, `rate`, and resets `loop_start`/`loop_end`.
- **Calls:** `Decoder::Get()`, `decoder->Frames()`, `decoder->BytesPerFrame()`, `decoder->Decode()`, `decoder->IsSixteenBit()`, `decoder->IsStereo()`, `decoder->IsSigned()`, `decoder->IsLittleEndian()`, `decoder->Rate()`
- **Notes:** Uses `auto_ptr` for automatic decoder cleanup. Calls `Clear()` on decode failure to free allocated buffer. Fixed-point math for sample rate (`FIXED_ONE * decoder->Rate()`).

### SoundReplacements::GetSoundOptions
- **Signature:** `SoundOptions* GetSoundOptions(short Index, short Slot)`
- **Purpose:** Retrieve sound options by audio type and variant; implements hash-table lookup with linear-search fallback.
- **Inputs:** `Index` ΓÇô sound type; `Slot` ΓÇô sound variant
- **Outputs/Return:** Pointer to `SoundOptions` if found; `0` (null) if not found
- **Side effects:** Updates hash table entry (`HashVal`) on successful linear search.
- **Calls:** `SoundOptions::HashFunc()`, vector dereferencing and iteration
- **Notes:** Two-stage lookup: fast-path checks hash table; fallback performs linear search and updates hash for future queries. Avoids rehashing on miss.

### SoundReplacements::Add
- **Signature:** `void Add(const SoundOptions& Data, short Index, short Slot)`
- **Purpose:** Insert or update sound options identified by (Index, Slot).
- **Inputs:** `Data` ΓÇô sound options to store; `Index`, `Slot` ΓÇô sound identity
- **Outputs/Return:** None
- **Side effects:** Modifies `SOList`; creates new entry or overwrites existing.
- **Calls:** vector iteration, `push_back()`
- **Notes:** Always performs linear search to detect duplicates; does not update hash table on insert (relies on miss-based lazy update in `GetSoundOptions`).

### SoundReplacements::SoundReplacements (constructor)
- **Signature:** `SoundReplacements()` (private)
- **Purpose:** Initialize the singleton instance.
- **Side effects:** Resizes `SOHash` to `SoundOptions::HashSize` (256) with `NONE` sentinels.

## Control Flow Notes
- **Init:** Singleton created lazily via `instance()` on first access.
- **Load:** External sounds loaded via `LoadExternal()`, which decodes the file and populates audio metadata.
- **Access:** `GetSoundOptions()` is the lookup entry point; uses hash table for O(1) average case, with O(n) linear fallback on miss.
- **Update:** `Add()` inserts or overwrites entries; hash table is not eagerly updated (deferred to `GetSoundOptions()` on next miss).
- **Reset:** `Reset()` clears `SOList`; hash table is not cleared (stale entries will trigger re-search).

## External Dependencies
- **Decoder.h** ΓÇô `Decoder::Get()` factory; query methods (`Frames()`, `BytesPerFrame()`, `Decode()`, `IsSixteenBit()`, etc.)
- **SoundFile.h** ΓÇô `SoundHeader` base class (`Load()`, `Clear()` methods)
- **FileHandler.h** ΓÇô `FileSpecifier` type
- **cseries.h** ΓÇô Low-level types (`int32`, `uint8`, `NONE`, `FIXED_ONE`)
- **C++ std** ΓÇô `std::vector`, `std::auto_ptr` (deprecated C++98 idiom; should be `std::unique_ptr` in modern C++)
