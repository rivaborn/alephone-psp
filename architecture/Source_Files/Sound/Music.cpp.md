# Source_Files/Sound/Music.cpp

## File Purpose
Implements singleton music playback manager for intro and level music in the Aleph One game engine. Handles decoder setup, audio buffering, fade-out effects, and level music playlists, delegating actual mixing to the Mixer component.

## Core Responsibilities
- Load and manage music files via StreamDecoder and Mixer integration
- Maintain intro music and level music playback state
- Implement fade-out with time-based volume interpolation
- Manage level music playlists with sequential or random song selection
- Fill audio buffers with decoded frames during playback
- Handle platform-specific (macOS) interrupt-safe buffer updates
- Coordinate initialization, volume checking, and playback state with SoundManager

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| StreamDecoder | external class | Decodes music file formats into PCM audio |
| FileSpecifier | external class | Represents a filesystem path to a music file |
| GM_Random | external class | Pseudorandom number generator for shuffle mode |
| std::vector\<uint8\> | vector | Decoded audio frame buffer (MUSIC_BUFFER_SIZE bytes) |
| std::vector\<FileSpecifier\> | vector | Playlist of level music file paths |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| m_instance | static Music* | static (singleton) | Singleton instance pointer |
| music_initialized | bool | instance | Whether a decoder is loaded and ready |
| music_intro | bool | instance | Whether intro music is active |
| music_play | bool | instance | Whether music should play (vs. paused) |
| music_prelevel | bool | instance | Flag: schedule Play() on next Idle() |
| music_level | bool | instance | Whether level music is active |
| music_fading | bool | instance | Whether fade-out is in progress |
| music_fade_start | uint32 | instance | SDL tick timestamp when fade began |
| music_fade_duration | uint32 | instance | Fade duration in milliseconds |
| decoder | StreamDecoder* | instance | Pointer to active audio decoder (NULL if none) |
| sixteen_bit, stereo, signed_8bit, bytes_per_frame, rate, little_endian | format flags | instance | Audio format metadata extracted from decoder |
| music_buffer | std::vector\<uint8\> | instance | Reusable decode buffer |
| music_file, music_intro_file | FileSpecifier | instance | Current and intro file paths |
| playlist | std::vector\<FileSpecifier\> | instance | Queue of level music files |
| song_number | size_t | instance | Current index in playlist (incremented per GetLevelMusic) |
| random_order | bool | instance | If true, randomize playlist order |
| randomizer | GM_Random | instance | Seeded RNG for shuffle mode |
| macos_file_done, macos_read_more, macos_buffer_length, macos_music_buffer | macOS-specific | instance | Platform-specific buffer state (macOS interrupt handling) |

## Key Functions / Methods

### Open(FileSpecifier *file)
- **Signature:** `void Open(FileSpecifier *file)`
- **Purpose:** Open and initialize a music file for playback; if already open and same file, rewind instead; if different file, close first.
- **Inputs:** Pointer to FileSpecifier (may be NULL)
- **Outputs/Return:** None
- **Side effects:** Calls Load(), sets music_initialized, stores music_file, resets macOS interrupt flags
- **Calls:** Load(), Rewind(), Close()
- **Notes:** If file is NULL, does nothing; Rewind() called instead of re-Load if file unchanged

### SetupIntroMusic(FileSpecifier &file)
- **Signature:** `bool SetupIntroMusic(FileSpecifier &file)`
- **Purpose:** Store intro music file and open it; set intro flag if successful
- **Inputs:** Reference to FileSpecifier
- **Outputs/Return:** Boolean indicating success (music_initialized)
- **Side effects:** Calls Open(), sets music_intro flag
- **Calls:** Open()

### FadeOut(short duration)
- **Signature:** `void FadeOut(short duration)`
- **Purpose:** Initiate a linear fade-out over specified duration (in ms); only works if music_play is true
- **Inputs:** Duration in milliseconds
- **Outputs/Return:** None
- **Side effects:** Sets music_fading=true, captures current SDL_GetTicks(), clears music_play flag if not level music
- **Calls:** SDL_GetTicks()

### Idle()
- **Signature:** `void Idle()`
- **Purpose:** Main per-frame update: process prelevel flag, restart if stopped, handle fade-out volume ramps
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Updates music_prelevel flag, calls Play/Restart, calls Mixer::SetMusicChannelVolume during fade, calls FillBuffer on macOS
- **Calls:** Playing(), Restart(), SDL_GetTicks(), GetVolumeLevel(), Mixer::SetMusicChannelVolume(), FillBuffer(), Pause()
- **Notes:** Fade volume calculated as `max_vol - (elapsed * max_vol) / duration`; clamps to [0, max_vol]

### Load(FileSpecifier &song_file)
- **Signature:** `bool Load(FileSpecifier &song_file)`
- **Purpose:** Create decoder instance and extract audio format metadata (sample rate, bit depth, stereo/mono, endianness)
- **Inputs:** Reference to FileSpecifier
- **Outputs/Return:** Boolean (true if decoder created successfully)
- **Side effects:** Deletes old decoder, creates new StreamDecoder, sets sixteen_bit/stereo/signed_8bit/bytes_per_frame/rate/little_endian
- **Calls:** StreamDecoder::Get(), decoderΓåÆIsSixteenBit/IsStereo/IsSigned/BytesPerFrame/Rate/IsLittleEndian, Mixer::obtained.freq
- **Notes:** Rate is calculated as fixed-point ratio: `(decoder_rate / mixer_output_rate) * 2^FIXED_FRACTIONAL_BITS`

### Play()
- **Signature:** `void Play()`
- **Purpose:** Start music playback by filling buffer and passing to Mixer
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls FillBuffer(), calls Mixer::StartMusicChannel(), calls CheckVolume()
- **Calls:** FillBuffer(), Mixer::StartMusicChannel(), CheckVolume()
- **Notes:** Only proceeds if music_initialized and SoundManager is active

### FillBuffer()
- **Signature:** `bool FillBuffer()`
- **Purpose:** Decode next frame of audio data into music_buffer; handle macOS special case; signal underrun on failure
- **Inputs:** None
- **Outputs/Return:** Boolean (true if bytes_read > 0)
- **Side effects:** Calls decoderΓåÆDecode(), updates macOS state flags, calls Mixer::UpdateMusicChannel()
- **Calls:** decoderΓåÆDecode(), GetVolumeLevel(), Mixer::UpdateMusicChannel()
- **Notes:** Returns false if volume is 0 or decoder is NULL; on macOS, FillBuffer does not update Mixer directly (deferred to InterruptFillBuffer)

### GetLevelMusic()
- **Signature:** `FileSpecifier* GetLevelMusic()`
- **Purpose:** Retrieve next song from playlist; apply random order if enabled; wrap-around on sequential mode
- **Inputs:** None
- **Outputs/Return:** Pointer to FileSpecifier in playlist (NULL if empty)
- **Side effects:** Increments song_number (or sets randomly); modifies randomizer state if random_order is true
- **Calls:** randomizer.KISS()

### CheckVolume()
- **Signature:** `void CheckVolume()`
- **Purpose:** Update Mixer music channel volume based on SoundManager::parameters.music and MAXIMUM_SOUND_VOLUME scale
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls Mixer::SetMusicChannelVolume()
- **Calls:** GetVolumeLevel(), Mixer::SetMusicChannelVolume()
- **Notes:** Skipped if music_fading (fade handler updates volume)

### Pause(), Close(), Rewind(), RestartIntroMusic(), Restart(), PreloadLevelMusic(), StopLevelMusic(), LoadLevelMusic(), SeedLevelMusic(), InterruptFillBuffer()
- Trivial state management / flag updates; see header for signatures

## Control Flow Notes
**Initialization:**  
Constructor initializes all flags (false), buffers, and singleton instance via static instance() method.

**Intro Playback:**  
SetupIntroMusic() ΓåÆ Open() ΓåÆ Play() ΓåÆ Idle() monitors Playing() and calls Restart() if stopped; RestartIntroMusic() restarts the same file.

**Level Playback:**  
PreloadLevelMusic() loads first song and sets music_prelevel flag ΓåÆ next Idle() detects flag and calls Play() ΓåÆ during playback, Idle() fills buffers via FillBuffer() ΓåÆ on decoder underrun, GetLevelMusic() fetches next song or repeats; StopLevelMusic() clears flags and closes.

**Fade-Out:**  
FadeOut() sets fade flags and timestamp ΓåÆ Idle() interpolates volume from max down to 0 over duration ΓåÆ calls Pause() when elapsed time exceeds duration.

**macOS Special Case:**  
InterruptFillBuffer() is called from Mixer callback at interrupt time and copies music_buffer to macOS-specific buffer; FillBuffer() is called at main thread time and sets macos_read_more flag to signal decoding on next idle.

## External Dependencies
- **Notable includes:** Music.h, Mixer.h, XML_LevelScript.h
- **SDL:** SDL_GetTicks() (timing), SDL_RWops* (unused in this file)
- **StreamDecoder:** Decoder::Get(), decoderΓåÆDecode()/Rewind()/IsSixteenBit/IsStereo/IsSigned/BytesPerFrame/Rate/IsLittleEndian
- **FileSpecifier:** File path wrapper; equality comparison
- **SoundManager:** instance(), IsInitialized(), IsActive(), parameters.music, GetNetmicVolumeAdjustment()
- **Mixer:** instance(), MusicPlaying(), StartMusicChannel(), UpdateMusicChannel(), SetMusicChannelVolume(), StopMusicChannel(), obtained.freq
- **GM_Random:** randomizer.KISS(), randomizer.SetTable()
- **XML_LevelScript:** Not directly called from this file (included but unused)
