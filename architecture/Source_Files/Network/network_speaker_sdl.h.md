# Source_Files/Network/network_speaker_sdl.h

## File Purpose
Defines the interface between SDL network audio receiving code and SDL sound playback routines for real-time microphone playback in networked Marathon: Aleph One games. Acts as a bridge layer abstracting buffer management between the network speaker subsystem and audio output.

## Core Responsibilities
- Define buffer descriptor structure for network audio data transmission
- Declare flag enumeration for buffer lifecycle management (disposable marking)
- Export dequeue function for sound playback to retrieve incoming network audio
- Export release function to return buffers to the free pool
- Provide inline helper to check buffer disposal requirement

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `NetworkSpeakerSoundBufferDescriptor` | struct | Wraps network audio data: pointer, length, and lifecycle flags |
| `kSoundDataIsDisposable` | enum constant | Flag (0x01) marking whether caller must release buffer via `release_network_speaker_buffer()` |

## Global / File-Static State
None.

## Key Functions / Methods

### dequeue_network_speaker_data
- Signature: `NetworkSpeakerSoundBufferDescriptor* dequeue_network_speaker_data()`
- Purpose: Fetch the next buffered network audio packet for playback
- Inputs: None
- Outputs/Return: Pointer to descriptor containing audio data; pointer invalidated on next call
- Side effects: Dequeues one buffer from incoming network audio queue
- Calls: Not defined in this file
- Notes: Also called by main thread in `close_network_speaker()` during shutdown; caller must check `kSoundDataIsDisposable` flag to determine if `release_network_speaker_buffer()` is required

### release_network_speaker_buffer
- Signature: `void release_network_speaker_buffer(byte* inBuffer)`
- Purpose: Return an audio buffer to the free pool after playback
- Inputs: `inBuffer` ΓÇö pointer to data previously returned by `dequeue_network_speaker_data()`
- Outputs/Return: None
- Side effects: Marks buffer available for reuse in network receive pipeline
- Calls: Not defined in this file
- Notes: Only called when `is_sound_data_disposable()` returns true for the corresponding descriptor

### is_sound_data_disposable
- Signature: `__inline__ bool is_sound_data_disposable(NetworkSpeakerSoundBufferDescriptor* inBuffer)`
- Purpose: Query whether caller must release a descriptor's buffer
- Inputs: `inBuffer` ΓÇö pointer to buffer descriptor
- Outputs/Return: Boolean (true if `kSoundDataIsDisposable` flag is set)
- Side effects: None
- Calls: None
- Notes: Inline for zero-cost abstraction of flag bit logic

## Control Flow Notes
Fits into the audio playback frame: sound engine periodically calls `dequeue_network_speaker_data()` to pull incoming network microphone streams, renders audio, then conditionally calls `release_network_speaker_buffer()` based on disposability check. Decouples network receiver thread (enqueueing) from audio playback thread (dequeueing).

## External Dependencies
- `#include "cseries.h"` ΓÇö platform abstraction; provides `byte`, `uint32`, SDL integration
- Implementation of exported functions: defined elsewhere (likely `network_speaker_sdl.c`)
