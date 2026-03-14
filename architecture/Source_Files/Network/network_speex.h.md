# Source_Files/Network/network_speex.h

## File Purpose
Header declaring Speex audio codec state management for network audio transmission in Aleph One (Marathon game engine). Provides initialization and cleanup routines for encoding/decoding game audio over the network using the Speex codec library.

## Core Responsibilities
- Declare global encoder/decoder state objects and bit buffers
- Export initialization functions for encoder and decoder setup
- Export cleanup functions for graceful shutdown
- Conditionally compile audio codec support based on SPEEX define

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpeexBits` | struct (external) | Bit-level buffer management for Speex encoder/decoder I/O |
| `SpeexPreprocessState` | opaque type (external) | Preprocessing state for audio signal conditioning |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gEncoderState` | `void*` | global | Opaque handle to Speex encoder codec state |
| `gEncoderBits` | `SpeexBits` | global | Bit stream buffer for encoder output |
| `gDecoderState` | `void*` | global | Opaque handle to Speex decoder codec state |
| `gDecoderBits` | `SpeexBits` | global | Bit stream buffer for decoder input |
| `gPreprocessState` | `SpeexPreprocessState*` | global | Audio preprocessing state (noise reduction, etc.) |

## Key Functions / Methods

### init_speex_encoder
- Signature: `void init_speex_encoder()`
- Purpose: Initialize the global Speex encoder state and bit buffer
- Inputs: None
- Outputs/Return: None (modifies global state)
- Side effects: Allocates and initializes `gEncoderState` and `gEncoderBits`
- Calls: (defined elsewhere)
- Notes: Must be called before encoding audio; paired with `destroy_speex_encoder()`

### destroy_speex_encoder
- Signature: `void destroy_speex_encoder()`
- Purpose: Free encoder resources and reset global state
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates `gEncoderState`, clears `gEncoderBits`
- Calls: (defined elsewhere)
- Notes: Called during shutdown; assumes `init_speex_encoder()` was previously called

### init_speex_decoder
- Signature: `void init_speex_decoder()`
- Purpose: Initialize the global Speex decoder state and bit buffer
- Inputs: None
- Outputs/Return: None (modifies global state)
- Side effects: Allocates and initializes `gDecoderState` and `gDecoderBits`
- Calls: (defined elsewhere)
- Notes: Must be called before decoding audio; paired with `destroy_speex_decoder()`

### destroy_speex_decoder
- Signature: `void destroy_speex_decoder()`
- Purpose: Free decoder resources and reset global state
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates `gDecoderState`, clears `gDecoderBits`
- Calls: (defined elsewhere)
- Notes: Called during shutdown; assumes `init_speex_decoder()` was previously called

## Control Flow Notes
This module is part of network audio initialization/shutdown. Encoder and decoder are likely initialized early in network subsystem setup and destroyed during shutdown. The preprocessing state (`gPreprocessState`) suggests audio signal conditioning (e.g., noise suppression) applied before encoding.

## External Dependencies
- `cseries.h` ΓÇö cross-platform compatibility layer (SDL, macOS emulation)
- `<speex/speex.h>` ΓÇö Speex codec library (encoder/decoder core API)
- `<speex/speex_preprocess.h>` ΓÇö Speex audio preprocessing (noise reduction)
