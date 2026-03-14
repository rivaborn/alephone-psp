# Source_Files/Misc/vbl_definitions.h

## File Purpose
Header file that defines data structures and prototypes for the replay recording/playback system and timer task management. Used exclusively by `vbl.c` and `vbl_macintosh.c` to manage vertical-blank (VBL) interrupt-driven recording of player actions and game state.

## Core Responsibilities
- Define action queue structure for buffering player input during recording
- Define replay metadata header and private replay state management structure
- Declare timer task installation/removal for VBL-driven callbacks
- Provide accessor macros/functions for player recording queues
- Declare preference key mapping function

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `ActionQueue` | struct | Circular queue buffer (read/write indices + data pointer) for queuing player actions during replay recording |
| `recording_header` | struct | Replay metadata: duration, player count, level number, game state checksum, version, player start positions, game data snapshot (352 bytes total) |
| `replay_private_data` | struct | Complete replay state: header, playback speed, recording/playback flags, action queues, file cache, and embedded resource data |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `replay` | `struct replay_private_data` | extern/global | Active replay state used across recording and playback operations |

## Key Functions / Methods

### install_timer_task
- **Signature:** `timer_task_proc install_timer_task(short tasks_per_second, bool (*func)(void))`
- **Purpose:** Register a periodic callback to fire at VBL (vertical blank) interrupts
- **Inputs:** `tasks_per_second` (frequency), `func` (callback returning bool)
- **Outputs/Return:** Opaque timer task handle for later removal
- **Side effects:** Installs system interrupt handler; captures per-frame callback
- **Calls:** Not inferable from this file (platform-specific implementation)
- **Notes:** Likely used to capture input every VBL for consistent replay granularity

### remove_timer_task
- **Signature:** `void remove_timer_task(timer_task_proc proc)`
- **Purpose:** Unregister and clean up a previously installed timer task
- **Inputs:** Handle from `install_timer_task`
- **Outputs/Return:** None
- **Side effects:** Removes interrupt handler
- **Calls:** Not inferable
- **Notes:** Must be paired with `install_timer_task`

### get_player_recording_queue
- **Signature:** `ActionQueue *get_player_recording_queue(short player_index)` (or macro: `replay.recording_queues + (x)`)
- **Purpose:** Retrieve the action queue for a specific player
- **Inputs:** `player_index` (0-based player number)
- **Outputs/Return:** Pointer to that player's `ActionQueue`
- **Side effects:** None (read-only in non-DEBUG builds; function call in DEBUG)
- **Notes:** Macro in release, bounds-checked function in debug builds

### set_keys_to_match_preferences
- **Signature:** `void set_keys_to_match_preferences(void)`
- **Purpose:** Apply user key binding preferences to the engine
- **Inputs:** None (reads from preferences store)
- **Outputs/Return:** None
- **Side effects:** Updates global key mapping state
- **Calls:** Not inferable

## Control Flow Notes
This file provides infrastructure for the replay/recording subsystem. The action queues capture input every VBL interrupt; the recording header and replay state manage persistence and playback. Integration point is likely at frame/update time, where timer callbacks enqueue actions and the replay system dequeues them during playback.

## External Dependencies
- **Notable types:** `player_start_data`, `game_data` (defined elsewhere; included implicitly)
- **Platform support:** References `vbl_macintosh.c`, suggesting macOS/classic Mac timing; typedef `timer_task_proc` is opaque (platform-specific)
- **Constants:** `MAXIMUM_QUEUE_SIZE` (512), `SIZEOF_recording_header` (352 bytes)
