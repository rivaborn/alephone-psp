# Source_Files/Network/network_microphone_sdl_alsa.cpp

## File Purpose
Implements network microphone audio capture on Linux using ALSA (Advanced Linux Sound Architecture). Provides initialization, hardware/software configuration, and async callback-driven frame capture with optional Speex compression support. Conditionally compiled only when `HAVE_ALSA` is defined; falls back to dummy implementation otherwise.

## Core Responsibilities
- Initialize ALSA PCM device and configure for 8kHz mono 16-bit capture
- Allocate and set hardware/software parameters (access mode, format, sample rate, channels, period size)
- Open and close ALSA capture device handle
- Register/unregister async callback handler for frame-ready events
- Control microphone capture state (start/stop)
- Read available audio frames and queue for network transmission
- Handle buffer underrun recovery and optional Speex encoder initialization
- Report microphone availability on this platform

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `snd_pcm_t` | opaque struct (ALSA) | Audio device handle for capture stream |
| `snd_pcm_hw_params_t` | opaque struct (ALSA) | Hardware parameter configuration |
| `snd_pcm_sw_params_t` | opaque struct (ALSA) | Software parameter configuration |
| `snd_async_handler_t` | opaque struct (ALSA) | Async event callback handler registration |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `capture_handle` | `snd_pcm_t*` | static | ALSA PCM device for audio input |
| `hw_params` | `snd_pcm_hw_params_t*` | static | Hardware parameters (freed after use) |
| `initialized` | `bool` | static | Tracks successful initialization |
| `active` | `bool` | static | Declared but unused |
| `bytes_per_frame` | `int` | static (const) | 2 (16-bit mono) |
| `frames` | `snd_pcm_uframes_t` | static | Period size in frames (derived from packet byte count) |
| `mic_active` | `bool` | static | Current capture active state |
| `pcm_callback` | `snd_async_handler_t*` | static | Registered async callback handler |

## Key Functions / Methods

### open_network_microphone
- Signature: `OSErr open_network_microphone()`
- Purpose: Initialize ALSA PCM device and validate audio format
- Inputs: None
- Outputs/Return: `0` on success; `-1` on any ALSA call failure
- Side effects:
  - Opens "default" ALSA capture device
  - Allocates, configures, and applies hardware parameters (16-bit signed LE/BE, 8kHz, mono, period-based access)
  - Allocates and configures software parameters (availability threshold, start threshold)
  - Calls `announce_microphone_capture_format()` to validate format with higher-level code
  - Conditionally calls `init_speex_encoder()` if `network_preferences->use_speex_encoder`
  - Sets `initialized = true`
  - Prints error messages to stderr on failure
- Calls: `snd_pcm_open()`, `snd_pcm_hw_params_malloc()`, `snd_pcm_hw_params_any()`, `snd_pcm_hw_params_set_access()`, `snd_pcm_hw_params_set_format()`, `snd_pcm_hw_params_set_rate_near()`, `snd_pcm_hw_params_set_channels()`, `snd_pcm_hw_params_set_period_size_near()`, `snd_pcm_hw_params()`, `snd_pcm_sw_params_malloc()`, `snd_pcm_sw_params_current()`, `snd_pcm_sw_params_set_avail_min()`, `snd_pcm_sw_params_set_start_threshold()`, `announce_microphone_capture_format()`, `init_speex_encoder()`
- Notes: Hard-coded 8kHz, mono, 16-bit format; returns on first error; endianness determined by `ALEPHONE_LITTLE_ENDIAN` macro

### close_network_microphone
- Signature: `void close_network_microphone()`
- Purpose: Release ALSA resources and shut down capture
- Inputs: None
- Outputs/Return: None
- Side effects:
  - Sets `initialized = false`
  - Closes PCM device via `snd_pcm_close()`
  - Nulls `capture_handle`
  - Conditionally calls `destroy_speex_encoder()` if `network_preferences->use_speex_encoder`
- Calls: `snd_pcm_close()`, `destroy_speex_encoder()`
- Notes: No error checking; assumes device is not actively capturing

### CaptureCallback
- Signature: `void CaptureCallback(snd_async_handler_t*)`
- Purpose: Async callback invoked when ALSA has audio frames ready
- Inputs: `snd_async_handler_t*` (unused context pointer from handler registration)
- Outputs/Return: None
- Side effects:
  - Reads available frames from ALSA buffer into static 16KB local buffer
  - Calls `copy_and_send_audio_data()` for each period worth of frames
  - On underrun (`-EPIPE`), recovers by calling `snd_pcm_prepare()`
  - Loops while available frames ΓëÑ period size
- Calls: `snd_pcm_avail_update()`, `snd_pcm_readi()`, `snd_pcm_prepare()`, `copy_and_send_audio_data()`
- Notes: Uses static buffer (not thread-safe if called concurrently); silently skips frames if `frames_read <= 0` (except -EPIPE)

### set_network_microphone_state
- Signature: `void set_network_microphone_state(bool inActive)`
- Purpose: Start or stop audio capture
- Inputs: `bool inActive` (true to start, false to stop)
- Outputs/Return: None
- Side effects:
  - **On transition to active:** calls `snd_pcm_prepare()`, registers `CaptureCallback()` via `snd_async_add_pcm_handler()`, calls `snd_pcm_start()`
  - **On transition to inactive:** deregisters callback via `snd_async_del_handler()`, stops capture via `snd_pcm_drop()`
  - Updates `mic_active` state flag
  - Prints errors to stderr; returns early if not initialized
- Calls: `snd_pcm_prepare()`, `snd_async_add_pcm_handler()`, `snd_pcm_start()`, `snd_async_del_handler()`, `snd_pcm_drop()`
- Notes: Only acts on state transitions; redundant calls to the same state are no-ops

### is_network_microphone_implemented
- Signature: `bool is_network_microphone_implemented()`
- Purpose: Report platform support for network microphone
- Inputs: None
- Outputs/Return: `true` (when compiled with `HAVE_ALSA`)
- Side effects: None
- Calls: None

### network_microphone_idle_proc
- Signature: `void network_microphone_idle_proc()`
- Purpose: Placeholder for idle-time maintenance (currently no-op)
- Inputs: None
- Outputs/Return: None
- Side effects: None

## Control Flow Notes
1. **Init phase:** Engine calls `open_network_microphone()` ΓåÆ ALSA device configured, format announced
2. **Active phase:** Engine calls `set_network_microphone_state(true)` ΓåÆ device prepared, async callback registered, capture started
3. **Capture loop:** ALSA invokes `CaptureCallback()` asynchronously ΓåÆ reads frames ΓåÆ calls `copy_and_send_audio_data()` (network layer handles transmission)
4. **Stop phase:** Engine calls `set_network_microphone_state(false)` ΓåÆ callback unregistered, capture dropped
5. **Shutdown:** Engine calls `close_network_microphone()` ΓåÆ resources released

Uses **conditional compilation** (`#ifdef HAVE_ALSA`). If `HAVE_ALSA` undefined, includes dummy implementation instead.

## External Dependencies
- `<alsa/asoundlib.h>` ΓÇö ALSA PCM and async APIs (Linux only)
- `"cseries.h"` ΓÇö Common types (`uint8`, `OSErr`), endianness macro (`ALEPHONE_LITTLE_ENDIAN`)
- `"network_speex.h"` (conditional `SPEEX`) ΓÇö Speex encoder/decoder
- `"preferences.h"` ΓÇö Global `network_preferences` struct
- `"network_microphone_shared.h"` ΓÇö Shared interface: `announce_microphone_capture_format()`, `copy_and_send_audio_data()`, `get_capture_byte_count_per_packet()`
- `"network_microphone_sdl_dummy.cpp"` ΓÇö Fallback dummy implementation (included at end if `HAVE_ALSA` undefined)
