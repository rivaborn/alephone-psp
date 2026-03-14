# Source_Files/Network/network_speaker_shared.cpp

## File Purpose
Handles receiving and processing network audio from remote players in multiplayer games. Manages audio decompression (Speex codec), team-based audio filtering, and muting of individual player microphones via UI-accessible controls.

## Core Responsibilities
- Receive incoming network audio data from remote players
- Decode Speex-compressed audio frames if enabled
- Enforce team-audio-only flag (squad-chat filtering)
- Maintain and toggle mute list for individual players
- Queue decompressed audio for playback to the local player
- Provide user feedback (screen messages) on mute/unmute actions

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `network_audio_header_NET` | struct | Network-marshalled audio header with fixed 8-byte format |
| `network_audio_header` | struct | Unpacked audio header with format flags and codec info |
| `sIgnoredPlayers` | std::set<short> | Tracks indices of players whose mics are currently muted |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sIgnoredPlayers` | std::set<short> | static | Persistent set of muted player indices; toggle-state storage for mic muting |

## Key Functions / Methods

### received_network_audio_proc
- **Signature:** `void received_network_audio_proc(void *buffer, short buffer_size, short player_index)`
- **Purpose:** Entry point called by the network distribution system when audio arrives from a remote player.
- **Inputs:** 
  - `buffer` ΓÇö raw network packet containing `network_audio_header_NET` followed by audio data
  - `buffer_size` ΓÇö total size of the buffer in bytes
  - `player_index` ΓÇö sender's player index
- **Outputs/Return:** None (void); side effect is audio queuing or silent drop
- **Side effects:** 
  - Checks mute list and team affiliation
  - Unpacks network header via `netcpy()`
  - If Speex enabled and mReserved==1: decodes variable-length Speex frames
  - Calls `queue_network_speaker_data()` to enqueue decoded PCM audio for playback
- **Calls:** `netcpy()`, `get_player_data()`, `speex_bits_read_from()`, `speex_decode_int()`, `queue_network_speaker_data()`, `screen_printf()` (indirectly via mute functions)
- **Notes:** 
  - Early return if player is in `sIgnoredPlayers`
  - Team-chat filtering: only plays audio if `kNetworkAudioForTeammatesOnlyFlag` is unset OR local and remote players are on the same team
  - Speex decoding loop processes up to 2048/160 = 12ΓÇô13 frames; each frame is 160 samples at 16-bit signed

### mute_player_mic
- **Signature:** `void mute_player_mic(short player_index)`
- **Purpose:** Toggle mute state for a specific player's microphone; provide user feedback.
- **Inputs:** `player_index` ΓÇö index of the player to mute/unmute
- **Outputs/Return:** None
- **Side effects:** 
  - Inserts or erases `player_index` from `sIgnoredPlayers`
  - Calls `screen_printf()` to display mute status message
- **Calls:** `sIgnoredPlayers.find()`, `sIgnoredPlayers.erase()`, `sIgnoredPlayers.insert()`, `screen_printf()`
- **Notes:** 
  - Acts as a toggle: if already muted, unmutes; if unmuted, mutes
  - UI feedback displays which action was taken (add/remove from ignore list)

### clear_player_mic_mutes
- **Signature:** `void clear_player_mic_mutes()`
- **Purpose:** Clear all muted players at once (e.g., on level transition or netgame exit).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Empties `sIgnoredPlayers` set
- **Calls:** `sIgnoredPlayers.clear()`
- **Notes:** No user feedback provided; typically called programmatically, not from UI

## Control Flow Notes
This file operates during the **frame/update** phase of the game loop. When network packets carrying audio arrive, the network distribution layer calls `received_network_audio_proc()` asynchronously (or during packet processing). The function then decides whether to play, decode, and queue the audio based on mute state and team settings. Muting is player-driven via UI (e.g., `mute_player_mic()` called from input handlers), while `clear_player_mic_mutes()` is likely called during shutdown or netgame teardown.

## External Dependencies
- **Includes:**
  - `cseries.h` ΓÇö platform/endianness definitions, standard types
  - `network_sound.h` ΓÇö `queue_network_speaker_data()` declaration
  - `network_data_formats.h` ΓÇö `network_audio_header_NET`, `netcpy()` overloads
  - `network_audio_shared.h` ΓÇö `network_audio_header` struct, audio format constants, flags
  - `player.h` ΓÇö `get_player_data()`, `local_player` extern
  - `shell.h` ΓÇö `screen_printf()` for UI feedback
  - `speex/speex.h`, `network_speex.h` (conditional, SPEEX flag) ΓÇö `gDecoderState`, `gDecoderBits`, Speex codec functions
  - `<set>` ΓÇö std::set container

- **Defined elsewhere:**
  - `local_player` ΓÇö global pointer to local player data structure
  - `gDecoderState`, `gDecoderBits` ΓÇö global Speex decoder state/bitstream (from `network_speex.h`)
  - `queue_network_speaker_data()` ΓÇö enqueues PCM for playback (defined in another network speaker implementation file)
  - `netcpy()` ΓÇö byte-order conversion utilities for network structures
