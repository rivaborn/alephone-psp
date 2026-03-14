# Source_Files/Network/network_sound.h

## File Purpose
Header file defining the main interface for network audio functionality (microphone input and speaker output) in the Aleph One engine. Provides unified SDL/Mac-compatible abstractions for capturing and playing back networked voice communication between game clients.

## Core Responsibilities
- Speaker system initialization, idle processing, and shutdown
- Queueing and playback of received network audio data
- Player microphone muting controls
- Microphone capability detection and activation
- Microphone input processing via idle callbacks
- Platform abstraction (SDL/Mac audio interfaces unified under this API)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### open_network_speaker
- Signature: `OSErr open_network_speaker()`
- Purpose: Initialize the network speaker system; called once at startup by main thread
- Inputs: None
- Outputs/Return: OSErr (error code; 0 = noErr on success)
- Side effects: Allocates/initializes speaker system state
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Must be paired with close_network_speaker(); do not call multiple times without intervening close

### close_network_speaker
- Signature: `void close_network_speaker()`
- Purpose: Shut down network speaker system; called at shutdown
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates speaker resources
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Cleanup complementary to open_network_speaker()

### received_network_audio_proc
- Signature: `void received_network_audio_proc(void *buffer, short buffer_size, short player_index)`
- Purpose: Receive and queue incoming network audio for playback; called by network routines
- Inputs: buffer (audio data), buffer_size (bytes), player_index (source player)
- Outputs/Return: None
- Side effects: Queues data for speaker playback
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Data is stored for asynchronous playback during idle processing

### network_speaker_idle_proc
- Signature: `void network_speaker_idle_proc()`
- Purpose: Main-thread idle callback to process and output queued speaker data
- Inputs: None
- Outputs/Return: None
- Side effects: Writes audio to speaker; processes playback queue
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Called per-frame by main thread

### open_network_microphone
- Signature: `OSErr open_network_microphone()`
- Purpose: Set up microphone capture system; called once at startup
- Inputs: None
- Outputs/Return: OSErr (0 = noErr on success)
- Side effects: Allocates/initializes microphone input resources
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: Do not call twice without intervening close_network_microphone()

### close_network_microphone
- Signature: `void close_network_microphone()`
- Purpose: Clean up microphone system; safe to call multiple times
- Inputs: None
- Outputs/Return: None
- Side effects: Deallocates microphone resources
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: Multiple calls safe (idempotent)

### set_network_microphone_state
- Signature: `void set_network_microphone_state(bool inActive)`
- Purpose: Activate or deactivate microphone audio capture
- Inputs: inActive (true = capture, false = silent)
- Outputs/Return: None
- Side effects: Starts/stops audio input streaming
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: Requires prior successful open_network_microphone()

### is_network_microphone_implemented
- Signature: `bool is_network_microphone_implemented()`
- Purpose: Check if microphone support is compiled/available
- Inputs: None
- Outputs/Return: bool (true = support exists, false = no microphone support)
- Side effects: None
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: Returns false with "no hope" of microphone functionality; true does not guarantee user has hardware

### network_microphone_idle_proc
- Signature: `void network_microphone_idle_proc()`
- Purpose: Main-thread idle callback to process microphone input
- Inputs: None
- Outputs/Return: None
- Side effects: Captures and queues audio input for network transmission
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: Called per-frame by main thread to service microphone buffers

### mute_player_mic / clear_player_mic_mutes
- Signatures: `void mute_player_mic(short player_index)`, `void clear_player_mic_mutes()`
- Purpose: Mute individual player microphones or clear all mutes
- Inputs: player_index (for mute_player_mic)
- Outputs/Return: None
- Side effects: Updates mute state; silences specified players during playback
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Called by game logic to suppress microphone output from specific players

### queue_network_speaker_data
- Signature: `void queue_network_speaker_data(byte* inData, short inLength)`
- Purpose: Queue raw audio data for playback (alternative to received_network_audio_proc)
- Inputs: inData (audio buffer), inLength (byte count)
- Outputs/Return: None
- Side effects: Appends data to speaker output queue
- Calls: (implementation in NETWORK_SPEAKER.C)
- Notes: Also available to callers outside network routines for custom audio injection

### quiet_network_speaker
- Signature: `void quiet_network_speaker(void)`
- Purpose: Stop/silence speaker output
- Inputs: None
- Outputs/Return: None
- Side effects: Clears playback queue and halts output
- Calls: (implementation in NETWORK_SPEAKER.C)

### has_sound_input_capability
- Signature: `bool has_sound_input_capability(void)`
- Purpose: More accurately determine if audio input is available
- Inputs: None
- Outputs/Return: bool (true = input device available)
- Side effects: None
- Calls: (implementation in NETWORK_MICROPHONE.C)
- Notes: More reliable than is_network_microphone_implemented() for actual hardware detection

## Control Flow Notes
**Initialization (startup):** Main thread calls `open_network_speaker()` and `open_network_microphone()`.

**Per-frame update:** Main thread calls `network_speaker_idle_proc()` and `network_microphone_idle_proc()` to service queues.

**Audio reception:** Network layer calls `received_network_audio_proc()` with incoming audio; data is queued for playback on next idle.

**Microphone activation:** Game logic calls `set_network_microphone_state(true)` to start capture; microphone idle proc streams data to network layer each frame.

**Shutdown:** `close_network_speaker()` and `close_network_microphone()` called at exit.

## External Dependencies
- **cseries.h**: Platform abstraction layer; provides OSErr type, SDL integration, and basic type definitions
- **NETWORK_SPEAKER.C**: Implementation of speaker system functions
- **NETWORK_MICROPHONE.C**: Implementation of microphone system functions
