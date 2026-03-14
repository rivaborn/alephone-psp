# Source_Files/Network/network_speex.cpp

## File Purpose
Wrapper for the Speex audio codec, managing encoder and decoder initialization for networked voice communication in Aleph One. Handles codec state setup, quality configuration, and audio preprocessing (noise reduction, automatic gain control).

## Core Responsibilities
- Initialize Speex encoder with fixed bitrate (quality 3 = 8 kbps) and complexity settings
- Initialize Speex decoder with enhancement mode enabled
- Set up audio preprocessing: denoise and AGC (automatic gain control)
- Manage global encoder/decoder state lifecycle and bit buffer allocation
- Provide cleanup routines for both encoder and decoder

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `SpeexBits` | typedef (Speex) | Bit stream buffer for codec data |
| `SpeexPreprocessState` | typedef (Speex) | Preprocessor state (denoise/AGC) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `gEncoderState` | `void*` | global | Active Speex encoder instance |
| `gEncoderBits` | `SpeexBits` | global | Encoder output bit stream buffer |
| `gPreprocessState` | `SpeexPreprocessState*` | global | Audio preprocessor for encoder |
| `gDecoderState` | `void*` | global | Active Speex decoder instance |
| `gDecoderBits` | `SpeexBits` | global | Decoder input bit stream buffer |

## Key Functions / Methods

### init_speex_encoder
- **Signature:** `void init_speex_encoder()`
- **Purpose:** Initialize Speex encoder and audio preprocessor; idempotent (checks if already initialized).
- **Inputs:** None (uses global state and constants).
- **Outputs/Return:** None (modifies globals).
- **Side effects:** Allocates `gEncoderState`, `gEncoderBits`, `gPreprocessState`; sets codec parameters.
- **Calls:** `speex_encoder_init()`, `speex_encoder_ctl()` (6 calls), `speex_bits_init()`, `speex_preprocess_state_init()`, `speex_preprocess_ctl()` (3 calls).
- **Notes:** Quality hardcoded to 3 (8 kbps target); complexity hardcoded to 4; sample rate from `kNetworkAudioSampleRate` (8000 Hz); AGC level set to 70% of max (32768 ├ù 0.7).

### destroy_speex_encoder
- **Signature:** `void destroy_speex_encoder()`
- **Purpose:** Clean up encoder and preprocessor; idempotent.
- **Inputs:** None (uses global state).
- **Outputs/Return:** None (modifies globals).
- **Side effects:** Deallocates `gEncoderState`, `gEncoderBits`, `gPreprocessState`; nulls pointers.
- **Calls:** `speex_encoder_destroy()`, `speex_bits_destroy()`, `speex_preprocess_state_destroy()`.
- **Notes:** Checks for NULL before freeing; safe to call multiple times.

### init_speex_decoder
- **Signature:** `void init_speex_decoder()`
- **Purpose:** Initialize Speex decoder; idempotent.
- **Inputs:** None (uses global state and constants).
- **Outputs/Return:** None (modifies globals).
- **Side effects:** Allocates `gDecoderState`, `gDecoderBits`; sets codec parameters.
- **Calls:** `speex_decoder_init()`, `speex_decoder_ctl()` (2 calls), `speex_bits_init()`.
- **Notes:** Enhancement mode enabled; sample rate from `kNetworkAudioSampleRate` (8000 Hz).

### destroy_speex_decoder
- **Signature:** `void destroy_speex_decoder()`
- **Purpose:** Clean up decoder; idempotent.
- **Inputs:** None (uses global state).
- **Outputs/Return:** None (modifies globals).
- **Side effects:** Deallocates `gDecoderState`, `gDecoderBits`; nulls pointers.
- **Calls:** `speex_decoder_destroy()`, `speex_bits_destroy()`.
- **Notes:** Checks for NULL before freeing.

## Control Flow Notes
Encoder and decoder operate independently as parallel pipelines. Initialization is lazy (guarded by null checks), typically called during network session setup. Preprocessing (denoise + AGC) is applied only on the encoder side before transmission. File is conditionally compiled under `#ifdef SPEEX` and `#if !defined(DISABLE_NETWORKING)`.

## External Dependencies
- **Speex codec library:** `<speex/speex_preprocess.h>` (preprocessing), `speex_encoder_*`, `speex_decoder_*`, `speex_bits_*` (from `network_speex.h` includes `<speex/speex.h>`)
- **Network audio parameters:** `kNetworkAudioSampleRate` from `network_audio_shared.h` (8000 Hz constant)
- **Standard includes:** `cseries.h` (project common header)
- **Conditional:** `preferences.h` included but not used in this file
