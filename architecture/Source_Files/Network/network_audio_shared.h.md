# Source_Files/Network/network_audio_shared.h

## File Purpose
Header defining shared data structures and constants for network audio (VoIP) functionality in Marathon: Aleph One. Provides a unified format specification between the network microphone capture and speaker playback code.

## Core Responsibilities
- Define the in-memory header structure for network audio packets
- Declare audio format flags (e.g., team-restricted distribution)
- Specify and document the current network audio encoding parameters (sample rate, bit depth, channels)
- Centralize audio format constants to ensure consistency across microphone and speaker implementations

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `network_audio_header` | struct | In-memory header for transmitted sound data; contains reserved field (for future format codes) and flags (e.g., team-only restriction) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `kNetworkAudioIsStereo` | const bool | file-static | False; defines mono-channel audio format |
| `kNetworkAudioIs16Bit` | const bool | file-static | True; defines 16-bit sample depth |
| `kNetworkAudioIsSigned8Bit` | const bool | file-static | False; unsigned samples if 8-bit (not currently used) |
| `kNetworkAudioSampleRate` | const int | file-static | 8000 Hz; defines sample rate |
| `kNetworkAudioBytesPerFrame` | const int | file-static | Computed as 2 (16-bit); bytes per audio sample |
| `kNetworkAudioForTeammatesOnlyFlag` | enum (0x01) | file-static | Flag in `mFlags` to restrict audio to team members only |

## Key Functions / Methods
None (header only; defines types and constants).

## Control Flow Notes
Not applicableΓÇöthis is a definition-only header. Used during network audio initialization and packet serialization/deserialization to validate and interpret transmitted audio data.

## External Dependencies
- `cseries.h` ΓÇö provides `uint32` type and platform abstraction macros

**Notes**: 
- The comment indicates microphone code currently hardcodes 11025 Hz unsigned 8-bit mono, conflicting with the stated 8000 Hz / 16-bit constants. Format negotiation/versioning via the reserved field or flags enum is suggested but not yet implemented.
- File authored Feb 1, 2003, extracted from `network_speaker_sdl.h`.
