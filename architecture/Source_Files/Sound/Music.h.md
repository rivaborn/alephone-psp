# Source_Files/Sound/Music.h

## File Purpose
Singleton manager for intro and level music playback in the Aleph One game engine. Handles audio stream decoding, buffer management, fade effects, and platform-specific audio I/O. Supports per-level music playlists with optional randomization.

## Core Responsibilities
- Music playback lifecycle (open, play, pause, stop, close, restart, rewind)
- Fade in/out effects with configurable durations
- Audio stream decoding and buffer filling for real-time playback
- Level music playlist management with random ordering
- Volume synchronization with SoundManager
- Platform-specific buffer handling (separate logic for macOS)
- Stream format detection (bit depth, channels, endianness, sample rate)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `FileSpecifier` | struct/class (external) | Represents audio file paths and handles cross-platform file access |
| `StreamDecoder` | abstract class (external) | Provides interface for decoding audio streams from various formats |
| `GM_Random` | struct (external) | Mersenne TwisterΓÇôbased RNG for shuffle mode in level music playlists |
| `SDL_RWops` | typedef (external) | SDL file I/O abstraction for stream reading |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_instance` | `Music*` | static | Singleton instance pointer |
| `MUSIC_BUFFER_SIZE` | `int` (const) | static | Platform-dependent: 64KB on macOS, 1KB on others |

## Key Functions / Methods

### instance()
- Signature: `static Music *instance()`
- Purpose: Retrieve or create singleton instance
- Inputs: None
- Outputs/Return: Pointer to Music singleton
- Side effects: Lazily allocates Music object on first call
- Calls: `new Music()` (constructor)
- Notes: Typical double-checked locking pattern (no lock visible; assumes single-threaded init)

### Open(FileSpecifier *file)
- Signature: `void Open(FileSpecifier *file)`
- Purpose: Open and begin decoding a music file
- Inputs: File path specifier
- Outputs/Return: None (success/failure inferred via `Initialized()` or `Playing()`)
- Side effects: Creates/replaces StreamDecoder, allocates buffers, may start playback
- Calls: `Load()` (private)
- Notes: Replaces current music if already playing

### FadeOut(short duration)
- Signature: `void FadeOut(short duration)`
- Purpose: Gradually reduce music volume over specified duration
- Inputs: Duration in milliseconds (inferred)
- Outputs/Return: None
- Side effects: Sets fade flags, records start time and duration
- Calls: None visible
- Notes: Fade state queried during `FillBuffer()` and `Idle()`

### FillBuffer()
- Signature: `bool FillBuffer()`
- Purpose: Decode audio data into `music_buffer` for output
- Inputs: None
- Outputs/Return: true if new data available, false on EOF or error
- Side effects: Advances decoder, updates buffer; may trigger fade or track transitions
- Calls: `decoder->Decode()`, `GetLevelMusic()`, `LoadLevelMusic()`
- Notes: Non-blocking; called per-frame. On macOS, `InterruptFillBuffer()` variant exists for async contexts

### PreloadLevelMusic() / StopLevelMusic() / ClearLevelMusic()
- Signature: `void PreloadLevelMusic() / StopLevelMusic() / void ClearLevelMusic()`
- Purpose: Manage level music playlist (load, stop, clear)
- Inputs: None
- Outputs/Return: None
- Side effects: Queues music files; on StopLevelMusic, closes current stream
- Calls: `Close()`, `playlist.clear()`
- Notes: `PushBackLevelMusic(FileSpecifier&)` and `LevelMusicRandom(bool)` configure playlist before `PreloadLevelMusic()` is called

### CheckVolume()
- Signature: `void CheckVolume()`
- Purpose: Synchronize internal volume with SoundManager settings
- Inputs: None
- Outputs/Return: None
- Side effects: May update fade state or output levels
- Calls: `GetVolumeLevel()`, `SoundManager::instance()->parameters.music`
- Notes: Likely called periodically in `Idle()`

## Control Flow Notes
The Music class integrates into the game loop via:
- **Initialization**: `instance()` creates singleton; `SetupIntroMusic()` or `PreloadLevelMusic()` + `Open()` start playback.
- **Per-frame**: `Idle()` performs fade, volume, and state checks; `FillBuffer()` decodes and feeds audio buffers to the platform's audio subsystem.
- **Shutdown**: `Close()` cleans up decoder and stops playback.

Playlist logic: Level music files are pushed to `playlist`, then `LoadLevelMusic()` picks the next song (random if `random_order=true`, sequential otherwise) when the current track ends (detected in `FillBuffer()`).

## External Dependencies
- **cseries.h**: Base types (`uint32`, `uint8`, `int16`, `_fixed`), macros, SDL includes
- **Decoder.h**: `StreamDecoder` abstract interface
- **FileHandler.h**: `FileSpecifier` file abstraction
- **Random.h**: `GM_Random` random number generator
- **SoundManager.h**: Access to global music volume via `SoundManager::instance()->parameters.music`
- **SDL.h** (via cseries): `SDL_RWops` file I/O
